# 15. 애플리케이션에 API 호출이 너무 많다
우리는 소프트웨어를 개발할 때 직접 만들거나 시간과 노력을 절약하기 위해 상용 라이브러리를 구매 혹은 오픈소스 사용을 고려하거나 플랫폼과 함께 제공되는 번들 라이브러리의 코드를 그대로 가져와서 사용하는 것을 고민한다.  
그러기 위해 많은것을 고려하고(안정적인지, 충분한 기능을 제공하는지 등...) 사용하기로 결정하고 나면 또 다른 문제가 나타날 수 있다.  
애플리케이션이 다른 누군가가 작성한 라이브러리를 반복 호출하기만 하는 것처럼 보일 수 있는데, 이런 코드를 어떻게 변경해야 할까?  
라이브러리를 사용하면 테스트가 필요없다고 오해하기 쉬운데, 실제로 의미있는 코드를 새로 작성하지 않기 때문이다.  
단지 메소드를 호출하기에 코드는 단순하다. 이런 상황에서 잘못될 만한 것이 있을까?  
대체로 레거시 프로젝트의 출발은 이처럼 단순한데, 시간이 지날수록 코드는 커져가고 일은 단순하지 않게 된다.  
무언가를 변경하려면 매번 어플리케이션을 실행해 정상적으로 동작중임을 확인해야 한다. 그리고 프로그래머는 딜레마에 빠지게 된다.  
자신이 모든 코드를 작성한 것이 아님에도 시스템을 유지해야 하기 때문에 변경 작업을 확신을 갖고 진행할 수 없다는 점이다.  
라이브러리 호출로 지저분해진 시스템은 여러가지 면에서 다루기 힘든데,  

1. 시스템 구조를 개선할 방법을 알기 어렵다. -> 단지 API 호출만 볼수 있기 때문이다.
2. API를 직접 소유하지 않기 때문이다. -> 인터페이스, 클래스, 메소드의 이름을 바꿀 수 없고, 코드의 다른부분에서도 이용하도록 메소드를 추가 할 수도 없다.

다음 예제는 메일링 리스트 서버로서 엉성하며 실제로 동작할지 여부를 알수 없다
```Java
import java.io.IOException;
import java.util.Properties;

import javax.mail.*;
import javax.mail.internet.*;

public class MailingListServer
{
  public static final String SUBJECT_MARKER = "[list]";
  public static final String LOOP_HEADER = "X-Loop";
  
  public static void main (String [] args) {
    if(args.length != 8) {
      System.err.println("Usage: java MailingList <popHost> " +
          "<smtpHost> <pop3user> <pop3password> " +
          "<smtpHost> <smtppassword> <listname> " +
          "<relayinterval>");
        return;
    }
    
    HostInformation host = new HostInformation(args[0], args[1], args[2], args[3], args[4], args[5]);
    String listAddress = args[6];
    int interval = new Integer(args[7]).intValue();
    Roster roster = null;
    
    try{
      roaster = new FileRoster("roster.txt");
    } catch(Exception e) {
      System.err.println("unable to open roster.txt");
      return;
    }
    
    try{
      do{
        try{
          Properties properties = System.getProperties();
          Session session = Session.getDefaultInstance(properties, null);
          Store store = session.getStore("pop3");
          store.connect(host.pop3Host, -1, host.pop3User, host.pop3Password);
          
          Folder defaultFolder = store.getDefalutFolder();
          if(defaultFolder == null) {
            System.err.println("Unable to open default folder");
            return;
          }
          
          Folder folder = defaultFolder.getFolder("INBOX");
          if(folder == null) {
            System.err.println("Unable to get: " + defaultFolder);
            return;
          }
          
          folder.open(Folder.READ_WRITE);
          process(host. listAddress, roster, session, store, folder);
        } catch(Exception e) {
          System.err.println(e);
          System.err.println("(retrying mail check)");
        }
        
        System.err.println(".");
        try{
          Thread.sleep(interval * 1000);
        } catch(InterruptedException e) {        
        }        
      } while(true);
    } catch(Exception e) {
      e.printStackTrace();
    }    
  }
  
  private static void process(HostInformation host, String listAddress, Roster roster, Session session, Store store, Folder folder) throws MessagingException{
    try {
      if(folder.getMessageCount() != 0) {
        Message[] messages = folder.getMessages();
        doMessage(host, listAddress, roster, session, folder, messages);
      }
    } catch(Exception e) {
      System.err.println("message handling error");
      e.printStackTrace(System.err);
    } finally {
      folder.close(true);
      store.close();
    }
  }
  
  private static void doMessage(HostInformation host, String listAddress, Roster roster, Session session, Folder folder, Message[] messages) throws MessagingException, AddressException, IOException, noSuchProviderException {
    FetchProfile fp = new FetchProfile();
    fp.add(FetchProfile.Item.ENVELOPE);
    fp.add()FetchProfile.Item.FLAGS;
    fp.add("X-Mailer");
    folder.fetch(messages, fp);
    
    for(int i=0;i<messages.length;i++) {
      Message message = messages[i];
      if(message.getFlags().contains(Flags.Flag.DELETED))
        continue;
      
      System.out.println("message receiverd: " + message.getSubject());
      
      if(!roster.containsOneOf(message.getFrom()))
        continue;
      
      MimeMessage forward = new MimeMessage(session);
      InternetAddress result = null;
      Address[] fromAddress = message.getFrom();
      
      if(fromAddress != null && fromAddress.length > 0)
        result = new InternetAddress(fromAddress[0].toString());
      
      InternetAddress from = result;
      forward.setForm(from);
      forward.setReplyTo(new Address[]{ newInternetAddress(listAddress) });
      forward.addRecipients(Message.RecipentType.To, listAddress);
      forward.addRecipients(Message.RecipentType.BCC, roster.getAddresses);
      
      String subject = message.getSubject();
      
      if(-1 == message.getSubject().indexOf(SUBJECT_MARKER))
        subject = SUBJECT_MARKER + " " + message.getSubject();
      
      forward.setSubject(subject);
      forward.setSentDate(message.getSentDate());
      forward.addHeader(LOOP_HEADER, listAddress);
      Object content = message.getContent();
      
      if(content instanceof Multipart)
        forward.setContent((Multipart)content);
      else
        forward.setText((String)content);
        
      Properties props = new Properties();
      props.put("mail.smtp.host", host.smtpHost);
      Session smtpSession = Session.getDefaultInstance(props, null);
      Transport transport = smtpSession.getTransport("smtp");
      transport.connect(host.smtpHost, host.smtpUser, host.smtpPassword);
      transport.sendMessage(forward, roster.getAddresses());
      message.setFlag(Flags.Flag.DELETED, true);
    }
  }  
  
}
```
그다지 길지 않지만 쉽게 이해할 수는 없는게, API와 관련없는 코드가 거의 없기 때문이다.  
이 코드를 개선하려면, 첫번째로 코드의 핵심 연산을 식별해야 한다. 다음과 같이 간단한 설명을 작성해보자
> 이 코드는 명령행에서 설정 정보를 읽고, 파일로부터 이메일 주소 목록을 읽는다. 이메일을 주기적으로 확인한다. 수신메일을 발견하면, 파일 내의 이메일 주소 각각에 해당 메일을 전달한다.

이 프로그램에서는 입출력을 주로 수행하지만 그외에도 코드 내에서 스레드를 실행하여 메일 확인을 하기도 하고 수신메일에 기반해 새로운 메시지를 만들기도 한다.  
이 코드의 책임을 구분해서 정리하자면
1. 수신 메시지를 받아서 시스템에 전달한다
2. 메일 메시지를 발송한다
3. 수신 메시지와 수신자 목록을 바탕으로 새로운 메시지를 작성한다
4. 대부분의 시간은 휴면 상태로 있다가 수신 메일을 확인하기 위해 주기적으로 깨어난다

책임들을 살펴보면 자바 메일 API에 대한 의존도가 다양한것을 확인할 수 있다.
1과 2는 전적으로 API에 묶여있고, 3은 메시지를 표현하는 클래스는 API의 일부지만 가짜 수신 메시지를 생성함으로서 이 책임을 독립적으로 테스트 할 수 있다.  
4는 메일처리와 관계없이 일정간격의 스레드가 필요할 뿐이다.  
다음은 책임들을 분리한 설계를 보여준다.
![KakaoTalk_20230220_171725226](https://user-images.githubusercontent.com/50142323/220050179-0e9d9a43-93b3-4096-b672-236e5860974b.jpg)
listDriver 클래스는 시스템을 구동하며 메일을 주기적으로 체크하는 스레드를 가지고 있다.  
MailReceiver 클래스에 메일을 확인하도록 지시하며, MailReceiver 클래스는 메일을 읽고 메시지를 하나씩 MailForwarder 클래스에 보낸다.  
MailForwarder 클래스는 메일링 리스트 내의 수신자마다 메시지를 작성한 후 MailSender를 통해 메일을 보낸다.  
MessageProcessor와 MailService 인터페이스 덕분에 클래스를 독립적으로 테스트할 수 있다.  
특히 실제 메일을 발송하지 않고도 테스트 하네스 내에서 MailService 클래스를 테스트할 수 있다. MailService 인터페이스를 구현하는 FakeMailSender 클래스를 만들면 된다.  

지금까지는 좋은 설계가 어떤것인지를 봤는데, 실제로는 어떻게 구현해야 할까? 기본적으로 두가지 접근 방법이 있다.
1. API를 포장한다.
2. 책임을 기반으로 추출한다.

API 포장과, 책임기반 추출 중 어느것을 선택해야 할까?  
API 포장은 다음과 같은 상황에 사용하면 좋다.  
 - API의 크기가 상대적으로 작은 경우
 - 서드파트 라이브러리에 대한 의존관계를 완전히 분리하고 싶은 경우
 - 현재 테스트 루틴이 없고 API를 통해 테스트할 수 없는 탓에 테스트 루틴을 작성할 수 없는 경우

책임 기반 추출은 다음과 같은 상황에 사용하면 좋다.  
 - API가 복잡한 경우
 - 안전하게 메소드를 추출하는 도구를 가지고 있거나, 수동으로 추출할 수 있다는 확신이 드는 경우

API 포장은 작업량이 좀 많지만, 서드파티 라이브러리에 대한 의존관계에서 벗어나고 싶을 때 매우 유용하며 가끔 사용할 필요성이 생기곤 한다.  
책임 기반 추출을 사용할 때는 상위 수준의 이름을 가진 메소드를 추출함으로서 API 코드와 직접 작성된 로직이 함께 추출될 수 있다.  
많은 개발팀이 이 두개의 기법을 동시에 사용한다.  
테스트에는 얇은 래퍼를 사용하고, 애플리케이션에 적절한 인터페이스를 제공하려면 상위 수준의 래퍼를 사용한다.


