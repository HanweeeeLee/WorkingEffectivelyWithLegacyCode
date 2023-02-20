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
inport java.io.IOException;
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

