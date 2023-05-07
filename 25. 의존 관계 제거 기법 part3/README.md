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
다음은 C++에서의 가상 함수 문제를 보여주는 예제이다.
```cpp
class Pager
{
public:
  Pager() {
    reset();
    form Connection();
  }
  virtual void form Connection() {
    assert(state == READY);
    //여기에 복잡한 하드웨어 제어코드
    ...
  }
  void sendMessage(const std::string& address, const std:: string& message) {
    formConnection();
    ...
  }
  ...
};
```
위 코드는 생성자 안에서 formConnection 메소드가 호출된다.  
formConnection메소드는 가상메소드로 선언됐으므로 단지 서브클래스화와 메소드 재정의 기법만 사용할 수 있을 것처럼 보인다. 확인해보자.
```cpp
class TestingPager : public Pager
{
public :
  virtual void formConnection() {
  }
};
TEST(messaging,Pager)
{
  TestingPager pager;
  pager.sendMessage("5551212", "Hey, wanna go to a party? XXXOOO");
  LONGS_EQUAL(OKAY, pager.getStatus());
}
```
sendMessage가 호출 됐을때 TestingPager::formConnection이 사용돼 올바르게 동작할 것이다.

### 단계
1. 대체하고자 하는 인스턴스 변수를 식별한다.

## 템플릿 재정의

## 텍스트 재정의

