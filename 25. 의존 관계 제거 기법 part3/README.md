# 25. 의존 관계 제거 기법 part3

## 의존 관계 밀어 내리기

이 기법은 골칫거리 의존관계를 클래스에 있는 나머지 의존관계와 분리하는데 도움을 줄 뿐 아니라, 테스트 하네스 안에서 작업하기 쉽게 해준다.  

```cpp
class OffMarketTradeValidator : public TradeValidator
{
private:
  Trade& trade;
  bool flag;
  void showMessage() {
    int status = AfxMessageBox(makeMessage(), MB_ABORTRETRYIGNORE);
    if (status == IDRETRY) {
      SubmitDialog dlg(this, "Press okay if this is a valid trade");
      dlg.DoModal();
      if(dlg.wasSubmitted()) {
        g_dispatcher.undoLastSubmission();
        flag = true;
      }
    }
    else
      if (status -- IDABORT) {
        flag = false;
      }
  }
public:
  OffMarketTradeValidator(Trade& trade) : trade(trade), flag(false) {}
  bool isValid() const{
    if (inRange(trade.getDate()) && validDestination(trade.destination) && inHours(trade))) {
      flag = true;
    }
    showMessage();
    return flag;
  }
  ...
};
```

의존관계 밀어내리기를 사용하려면 현재 클래스를 추상으로 만들어야 한다. 그 후에 새로운 배포용 클래스가 될 서브 클래스를 하나 생성하고 모든 골칫거리 의존관계를 그 클래스 안으로 밀어 내린다.  
그리고 그 클래스를 테스트할 때 그것의 메소드들을 사용할 수 있도록 기존 클래스를 서브클래스화 할 수 있다.  

```cpp
class OffMarketTradeValidator : public TradeValidator
{
protected:
  Trade& trade;
  bool flag;
  virtual void showMessage() = 0;
public:
  OffMarketTradeValidator(Trade& trade) : trade(trade), flag(false) {}
  bool isValid() const{
    if (inRange(trade.getDate()) && validDestination(trade.destination) && inHours(trade))) {
      flag = true;
    }
    showMessage();
    return flag;
  }
  ...
};
class WindowsOffMarketTradeValidator : public OffMarketTradeValidator
{
protected:
  virtual void showMessage() {
    int status = AfxMessageBox(makeMessage(), MB_ABORTRETRYIGNORE);
    if (status == IDRETRY) {
      SubmitDialog dlg(this, "Press okay if this is a valid trade");
      dlg.DoModal();
      if(dlg.wasSubmitted()) {
        g_dispatcher.undoLastSubmission();
        flag = true;
      }
    }
    else
      if (status -- IDABORT) {
        flag = false;
      }
  }
  ...
};
```
새로운 서브클래스(WindowsOffMarketValidator)로 밀어 내려진 사용자 인터페이스용 함수나 클래스를 가지고 있다면, 테스트를 위한 또 다른 서브클래스를 작성할 수 있다.  
이 또다른 서브클래스가 하는 일은 ShowMessage 동작을 무효화 하는 것이다.  
```cpp
class TestingOffMarketTradeValidator : public OffMarketTradeValidator
{
protected:
  virtual void showMessage() {}
};
```
이제 사용자 인터페이스와 아무런 의존 관계도 가지지 않고 테스트할 수 있는 클래스를 하나 갖게 됐다.  
상속성을 이렇게 사용하는 것이 이상적인 형태는 아니지만 이렇게 하는것은 클래스 로직의 일부를 테스 루틴 아래로 가져오는데 도움을 준다.  
OffMarketTradeValidator용 테스트 루틴이 있다면, 재시도 로직을 깨끗하게 하고 WindowsOffMarketTradeValidator로부터 그것을 끌어올릴 수 있다.  

### 단계
1. 테스트 하네스 안에 의존 관계 문제를 가지는 클래스를 빌드하려고 시도한다.
2. 빌드할 때 어떤 의존관계들이 문제를 야기하는지 식별한다.
3. 그 의존 관계의 특정 환경과 통신할 수 있는 이름을 가지는 새 서브클래스를 하나 생성한다.
4. 서명 유지에 신경을 쓰면서 나쁜 의존관계를 가지는 인스턴스 변수와 메소드를 새 서브클래스로 복사한다. 이떄 기존 클래스 안에 있던 메소드들을 protected 추상으로 만들고 기존 클래스는 추상으로 만든다.
5. 테스트 서브클래스를 하나 생성하고, 그 서브클래스를 인스턴스화할 수 있도록 테스트 루틴을 변경한다.
6. 테스트 루틴을 빌드해 새 클래스를 인스턴스화할 수 있는지 검증한다.

## 함수를 함수 포인터로 대체
C언어 에서 사용하는 기법이라 생략해도 될듯...

### 단계
1. 대체하고자 하는 함수의 선언을 찾는다.
2. 각 함수 선언 앞에 같은 이름을 갖는 함수 포인터를 생성하자.
3. 기존 함수 선언에 다른 이름을 만들어줘서 이제 막 선언한 함수 포인터의 이름과 같은 이름이 없도록 만든다.
4. C파일에 있는 포인터들이 옛 함수들의 주소를 가리키도록 초기화 한다.
5. 옛 함수의 본문을 찾을 수 있도록 빌드한다. 옛 함수 이름을 새 함수 이름으로 바꾼다.

## 전역 참조를 get 메소드로 대체

클래스 안의 전역 요소들에 대한 의존 관계들을 해결하는 방법으로 클래스의 모든 의존 관계들을 위한 get 메소드 도입을 고려할 수 있다.  
get 메소드를 가지고 있다면 적절한 값을 반환하도록 하기 위해 서브클래스화와 메소드 재정의 기법을 사용할 수 있다.  
다음은 자바로 된 예제이다.
```Java
public class RegisterSale
{
  public void addItem(Barcode code) {
    Item newItem = Inventory.getInventory().itemForBarcode(code);
    items.add(newItem);
  }
  ...
}
```
이 코드에서 Inventory 클래스에 전역적으로 접근할 수 있다. 자바에서 클래스 자체는 전역 객체며, 그 클래스의 일을 하려면 어떤 상태를 참조해야 한다.  
전역 참조를 get 메소드로 대체 기법을 사용하면 이러한 문제를 해결할 수 있을까?  
먼저 get 메소드를 작성하는 일부터 시작한다. 테스트하에 재정의 할 수 있도록 이것을 protected로 만들고 전역 요소에 대한 모든 참조를 get 메소드로 대체한다.
```Java
public class RegisterSale
{
  public void addItem(Barcode code) {
    Item newItem = getInventory().itemForBarcode(code);
    items.add(newItem);
  }
  protected Inventory getInventory() {
    return Inventory.getInventory();
  }
  ...
}
```
이제 테스트하는데 사용할 수 있는 Inventory 클래스의 서브클래스를 작성할 수 있다.  
Inventory 클래스는 싱글톤이므로 이 클래스의 생성자를 private이 아닌 protected로 만들어야 한다.
```Java
public class FakeInventory extends Inventory
{
  public Item itemForBarcode(Barcode code) {
    ...
  }
  ...
}
```
이제는 테스트 루틴 안에서 사용할 클래스를 작성한다.
```Java
class TestingRegisterSale extends RegisterSale
{
  Inventory inventory = new FakeInventory();
  protected Inventory getInventory() {
    return inventory;
  }
}
```

### 단계
1. 대체하고자 하는 전역 참조를 식별한다.
2. 전역 참조를 위한 get 메소드를 작성한다. 이떄 메소드의 접근 보호가 충분히 느슨해야 한다는 것을 명심하자. 그래야 나중에 서브클래스에 있는 get 메소드를 재정의할 수있다.
3. 전역 참조를 get 메소드에 대한 호출로 대체한다.
4. 테스트 서브클래스를 하나 생성하고 get 메소드를 재정의한다.

## 서브클래스화와 메소드 재정의

이 기법은 객체지향 프로그램에서 의존관계를 제거하는데 사용되는 주요 기법이다.  
이 기법의 주된 아이디어는 관심없는 동작을 무효화하거나 관심있ㅎ는 동작에 접근하기 위해 테스트의 관점에서 상속성을 이용할 수 있다는 것이다.  
작은 애플리케이션에 있는 하나의 메소드를 살펴보자
```Java
class MessageForwarder
{
  private Message createForwardMessage(Session session, Message message) throws MessagingException, IOException {
    MimeMessage forward = new MimeMessage(session);
    
    forward.setFrom(getFromAddress(message));
    forward.setReplyTo(
      new Address[] { new InternetAddress(listAddress) }
    );
    forward.addRecipients(Message.RecipientType.TO, listAddress);
    forward.addRecipients(Message.RecipientType.BCC, getMailListAddress());
    forward.setSubject(transformedSubject(message.getSubject()));
    forward.setSentDate(message.getSentDate());
    forward.addHeader(LOOP_HEADER, listAddress);
    
    buildForwardContent(message, forward);
    return forward;
  }
  ...
}
```
테스트할 때 MimeMEssage 클래스에 대한 의존관계를 갖고 싶지 않다고 가정해보자.  
MimeMessage에 대한 의존 관계를 분리하고자 한다면 createForwardMessage를 protected로 만들고, 단지 테스트를 위해 만든 새 서브클래스 안에서 재정의 하면 된다.  
```Java
class TestingMessageForwarder extends MessageForwarder
{
  protected Message createForwardMessage(Session session, Message message) {
    Message forward = newFakeMessage(message);
    return forward;
  }
  ...
}
```

### 단계
1. 감지하길 원하는 장소나 분리하고자 하는 의존 관계를 식별하라. 목적을 달성하기 위해 재정의할 수 있는 가장 작은 메소드 집합을 찾으려고 노력하라.
2. 모든 메소드가 재정의될 수 있도록 만들자. 이렇게 하는 방법은 프로그래밍 언어마다 차이가 있을 수 있다. C++에서는 메소드가 기존에 존재하지 않는다면 가상으로 만들어야 한다. 자바에서는 메소드가 final이 아닌 형태로 만들어져야 한다. 여러 .NET 언어에서는 명시적으로 메소드가 재정의되도록 만들어야 한다.
3. 당신이 사용하는 언어에서 필요로 한다면, 재정의하려는 메소드의 가시성을 조절해 서브클래스에서 재정의될 수 있게 만든다. 자바나 C#인 경우 서브클래스에서 재정의되려면 가시성이 최소한 protected는 돼야 한다. C++에서는 메소드가 private이라도 서브클래스에서 재정의 될 수 있다.
4. 해당 메소드들을 재정의하는 서브클래스를 작성하고, 테스트 하네스 안에서도 빌드할 수 있는지 검증한다.

## 인스턴스 변수 대체
생성자 안에서 객체를 생성하는 것은 문제가 될 수 있는데, 특히 테스트 루틴 안의 객체들에 의존하기 어려운 경우가 그렇다.  
대부분의 경우에는 팩토리 메소드 추출과 재정의 기법을 통해 문제를 해결할 수 있지만, 생성자 안에서 가상 함수 호출 재정의를 허용하지 않는다면 다른 대안을 찾아야 하는데, 그중 하나가 인스턴스 변수 대체이다.  

```cpp
class A
{
public:
  A() {
    someMethod();
  }
  virtual void someMethod() {
  }
};
class B : public A
{
  C *c;
public :
  B() {
    c = new C;
  }
  virtual void someMethod() {
    c.doSomething();
  }
};
```
위 코드는 클래스 A의 someMethod를 재정의하는 B의 someMethod를 가지고 있다.  
하지만 생성자 호출 순서를 기억해보면, 클래스 B 하나를 생성하면 B의 생성자에 앞서 A의 생성자가 호출된다.  
따라서 A의 생성자는 someMethod를 호출하게 되고 someMethod를 재정의하게 돼 결국 B에있는 someMethod가 사용된다.  
이 메소드는 C의 참조형으로 존재하는 doSomething을 호출하려 할것이다.  
하지만 실제로는 B클래스의 생성자가 아직 실행되지 않았으므로 이 객체는 초기화 되지 않았다.  
C++에서는 이러한 작은 보호 기법이 생성자 안에서의 동작을 대체하려는 시도를 미연에 방지한다. 다행히 몇가지 다른 대안이 있다.  
대체하려는 객체가 생성자 안에서 사용되지 않는다면, 의존 관계를 제거하기 위해 get 메소드 추출과 재정의 기법을 사용할 수 있다.  
해당 객체를 사용하고 싶고 다른 메소드가 호출되기 전에 그 메소드를 확실히 대체하고자 한다면, 인스턴스 변수 대체를 사용할 수 있다.
```cpp
BlendingPen::BlendingPen()
{
  setName("BlendingPen");
  m_param = ParameterFactory::createParameter("cm", "Fade", "Aspect Alter");
  m_param->addChoice("blend");
  m_param->addChoice("add");
  m_param->addChoice("filter");
  setParamByName("cm", "blend");
}
```
이 경우에 생성자는 팩토리를 통해 매개변수를 생성하고 있다.  
클래스에 또 다른 메소드를 추가하는 것을 꺼리지 않는다면 생성자에서 생성한 매개변수를 대체할 수 있다.
```cpp
void BlendingPen::supersedeParameter(Parameter *newParameter)
{
  delete m_param;
  m_param = newParameter;
}
```
테스트 루틴 안에서는 필요한 만큼 펜을 생성할 수 있고, 감지 객체에 그것들을 두고자 할때는 supersedeParameter를 호출할 수 있다.  
얼핏 봤을 떄 인스턴스 변수 대체는 객체를 적절한 자리에서 감지하는데 그다지 좋지 않은 방법처럼 보일 수 있다.  
하지만 C++에서 생성자 안의 로직이 너무 복잡해 생성자 매개변수화 기법을 사용하기가 어색할때는 인스턴스 변수 대체 기법이 최선의 방법일 수 있다.  
일반적으로 생성자 안에서 가상 호출을 허용하는 언어라면 팩토리 메소드 추출과 재정의 기법이 더 나은 선택이 될것이다.

### 단계
1. 대체하고자 하는 인스턴스 변수를 식별한다.
2. SupersedeXXX라는 이름을 가진 메소드를 생성한다. 이떄 XXX는 대체하고자 하는 변수의 이름을 가리킨다.
3. 해당 메소드에 필요한 코드를 작성한다. 이코드를 통해 이 변수의 인스턴스를 소멸시키고 새로운 값으로 설정할 수 있다. 변수가 참조였다면 클래스 내에 객체가 가리키는 다른참조가 존재하지 않는다는 점을 검증하자. 혹 다른 참조가 존재한다면, 대체 메소드 안에서 추가적인 작업을 해야 할것이다. 이 작업으로 해당 객체를 대체하면 안전하고 올바른 효과를 가져온다는 것을 확신할수 있다.

## 템플릿 재정의
새로운 언어의 특징은 새로운 기법들을 제공하기도 한다.  
예를들어 어떤 언어가 포괄형과 형에 별칭 붙이기 등의 기능을 제공한다면 템플릿 재정의라 볼리는 기법을 통해 의존관계를 제거할 수 있다.  
```cpp
//AsyncReceptionPort.h
class AsyncReceptionPort
{
private:
  CSocket m_socket;
  Packet m_packet;
  int m_segmentSize;
  ...
public:
  AsyncRecpetionPort();
  void Run();
  ...
};

//AsyncRecptionPort.cpp
void AsyncRecptionPort::Run() {
  for(int n = 0; n < m_segmentSize; ++n) {
    int bufferSize = m_bufferMax;
    if(n = m_segmentSize - 1)
      bufferSize = m_remainingSize;
      
    m_socket.receive(m_receiveBuffer, bufferSize);
    m_packet.mark();
    m_packet.append(m_receiveBuffer, bufferSize);
    m_packet.pack();
  }
  m_packet.finalize();
}
```
이러한 코드에서 메소드 안의 로직을 변경하고자 한다면 소켓을 통해 어떤 정보를 보내지 않는 한, 테스트 하네스 안에서 메소드를 실행할 수 없다는 사실과 마주치게 된다.  
C++ 에서는 AsyncReptionPort를 통상적인 클래스가 아닌 템플릿으로 만듦으로서 이러한 문제를 피해갈 수 있다.  
```cpp
//AsyncReceptionPort.h
template<typename SOCKET> class AsyncReceptionPortImpl
{
private:
  SOCKET m_socket;
  Packet m_packet;
  int m_segmentSize;
  ...
public:
  AsyncRecpetionPort();
  void Run();
  ...
};

template<typename SOCKET>
void AsyncRecptionPort::Run() {
  for(int n = 0; n < m_segmentSize; ++n) {
    int bufferSize = m_bufferMax;
    if(n = m_segmentSize - 1)
      bufferSize = m_remainingSize;
      
    m_socket.receive(m_receiveBuffer, bufferSize);
    m_packet.mark();
    m_packet.append(m_receiveBuffer, bufferSize);
    m_packet.pack();
  }
  m_packet.finalize();
}

typedef AsyncReceptionPortImpl<CSocket> AsyncReceptionPort;
```
이 기법의 장점중 하나는 코드베이스에 있는 모든 참조를 일일이 변경하는 대신에 typedef를 사용해 그 일을 할 수 있다는 것이다.  
포괄형은 가지고 있지만 typedef와 같은 형 별칭 붙이기 기능을 제공하지 않는 언어에서는 컴파일러 의존 기법을 사용해야만 한다.  

### 단계
1. 테스트하고자 하는 클래스 안에 있는 대체하고자 하는 특징을 식별한다.
2. 해당 클래스를 템플릿으로 바꾼다. 그러려면 그 클래스를 대체하고자 하는 변수들을 통해 매개변수화하고, 메소드의 본문 내용을 헤더 안으로 복사해야 한다.
3. 그 템플릿에 다른 이름을 붙인다. 기계적으로 이 작업을 하는 방법 중 하나는 원래 템플릿 이름에 Impl이라는 접미어를 추가하는 것이다.
4. 템플릿 정의 뒤에 typedef문을 추가한다. 이 작업은 기존 클래스 이름을 가지고 해당 템플릿의 원래 매개변수들로 정의함으로서 수행한다.
5. 테스트 파일에 템플릿 정의를 인클루드하고 새로운 형으로 해당 템플릿을 인스턴스화 한다. 그렇게 함으로서 테스트를 위해 필요한 형들을 대체할 수 있다.

## 텍스트 재정의
(ruby같은 인터프리터 언어에 해당하는 기법)  
최신 인터프리터 언어들은 의존 관계를 제거하는 매우 뛰어난 방법을 제공한다. 코드를 해석할 떄 즉석에서 메소드를 재정의하는 것이다.  
다음은 루비 언어의 예다.
```ruby
# Account.rb
class Account
  def report_deposit(value)
  ...
  end
  def deposit(value)
    @balance += value
    report_deposit(value)
  end
  def withdraw(value)
    @balance -= value
  end
end
```
테스트할떄 report_deposit을 실행하고 싶지 않다면, 테스트 파일에서 메소드를 재정의하고 그 뒤에 테스트를 배치할 수 있다.
```ruby
# AccountTest.rb
require "runit/testcase"
require "Account"

class Account
  def report_deposit(value)
  end
end

# 여기서부터 테스트 코드
class AccountTest < RUNIT::TestCase
  ...
end
```
여기서는 Account 클래스 전체가 아니라 report_deposit 메소드만 재정의 하고 있다는 점에 유의하자.  
루비 인터프리터는 루비 파일 내의 모든 행을 실행 가능한 문장으로서 해석한다.  
루비 인터프리터는 메소드 정의가 이미 존재하는지 여부에 신경 쓰지 않는다. 기존에 정의된 것이 존재한다면 그냥 교체할 뿐이다.  

### 단계
1. 대체하려는 정의를 포함하는 클래스를 식별한다.
2. 테스트용 소스 파일의 맨 위에 클래스를 포함하는 모듈 이름으로 require절을 추가한다.
3. 대체하려는 모든 메소드에 대해 테스트용 소스 파일의 맨 위에 대체 정의를 기술한다.

