# 10. 테스트 하네스에서 이 메소드를 실행할 수 없다
대부분의 경우, 클래스 생성은 단지 시작에 불과한데 그 다음 단계로는 변경 대상 메소드를 위한 테스트 루틴의 작성이다.  
메소드가 그다지 많은 인스턴스를 사용하지 않는다면 정적 메소드 드러내기 기법을 사용해 코드에 접근할 수 있고,  
메소드가 너무 길어 다루기 어렵다면 인스턴스 생성이 좀 더 쉬운 클래스로 코드를 이동하는 메소드 객체 추출 기법을 사용할 수 있다.  
다행히 대체로 메소드에 대한 테스트 루틴을 작성하기 위해 필요한 작업량은 그리 많지 않은데, 이 과정에서 발생할 수 있는 문제는 다음과 같다.  
 - 메소드를 테스트 루틴에서 접근할수 없는 경우 -> 메소드가 private으로 선언되었거나 그 밖의 가시성 문제가 있는 경우다.
 - 메소드 호출에 필요한 매개변수를 생성하기 어려워서 메소드를 호출하기 어려운 경우
 - 메소드에 부정적인 부작용(데이터베이스 변경 등...) 때문에 테스트 하네스 안에서 실행 불가능한 경우
 - 메소드가 사용하는 객체들을 사전에 감지해야 하는 경우
이번 장에서는 이런 문제들을 해결하는 기법들을 소개한다.

## 숨어있는 메소드
클래스 내의 메소드를 변경해야 하는데 private 메소드일 경우 어떻게 해야할까?  
우선 public 메소드를 통해 테스트 가능할지를 검토해봐야 하는데, 가능할거 같다면 시도해볼 가치가 있다.  
 - 고생해서 private 메소드에 접근할 필요가 없고, public 메소드를 통해 테스트하기에 실제 코드와 동일한 방법에 의한 테스트가 보장된다는 점
 - 작업량이 약간 줄어드는 것도 장점이다.

위와 같은 장점이 있지만 어떤경우에는 단순히 클래스 내 깊숙히 위치하는 메소드의 테스트 루틴을 작성하고 싶은 경우나, public 메소드를 이용한 테스트가 어려울 수 있는데,  
그렇다면 private 메소드를 위한 테스트 루틴은 어떻게 작성하는 것이 좋을까? 다행히 이에 대한 직접적인 해답이 있다.  
private 메소드를 테스트 해야 한다면 그 메소드를 public으로 만들어야 한다는 것이다.  
public으로 만드는 것이 꺼려진다면 이는 곧 클래스가 너무 많은 책임을 갖고 있음을 의미하며, 곧 클래스를 수정해야한다는 의미이다.  
실제로 private 메소드를 public 메소드로 만드는 것이 왜 고민되는 지를 보자면
 1. 이 메소드는 단지 유틸리티이며, 즉 호출 코드는 이 메소드를 신경쓰지 않는다.
 2. 호출코드에서 이 메소드를 직접 사용하는 경우, 클래스의 다른 메소드의 결과 값에 영향을 미칠 수 있다.

첫번째 이유는 심각하지 않다. 이 메소드를 다른 클래스로 옮기는 것이 나을지 검토할 필요는 있겠지만 클래스의 인터페이스에 추가적인 public 메소드의 존재를 허용할 수 있다.  
두번째 이유는 다소 심각하지만 해결책이 있다. private 메소드를 신규 클래스로 옮기는 것이다.  
신규 클래스로 옮긴 뒤, 이 메소드를 public으로 선언함으로서 기존 클래스에서 신규 클래스의 인스턴스를 생성 할 수 있다.  
이렇게 하면 이 메소드의 테스트가 가능해지고 설계도 개선된다.  
좋은 설계는 테스트 가능한 설계며, 테스트 불가능한 설계는 나쁜 설계이다. 하지만 적절한 테스트 루틴들이 충분히 존재하지 않는 상황에서는 신중을 기해야 한다.  
예제를 통해 어떻게 해결할 수 있을지 살펴보자

```Cpp
Class CCAImage
{
private:
  void setSnapRegion(int x, int y, int dx, int dy);
  ...
public:
  void snap();
  ...
};
```
이 CCAImage 클래스는 보안 시스템에서 사진을 찍을때 사용된다. 왜 이미지 클래스가 사진을 찍는지 의문을 가질 수 있지만 이것은 레거시 코드라는걸 기억하자.  
이 클래스는 카메라를 제어하고, 저수준의 C언어 API를 사용해 사진을 찍는 snap() 이라는 메소드를 갖고있다.  
피사체의 움직임에 따라 snap() 메소드는 setSnapRegion()을 반복 호출해 사진을 버퍼의 어느곳에 저장할지 결정한다.  
그런데 API가 변경되어 setSnapRegion() 메소드도 변경해야 하는 상황이라면 어떻게 해야할까?  
한가지 방법은 setSnapRegion()을 그냥 public으로 선언하는 것이다. 하지만 이렇게 되면 부정적인 결과가 나올수 있다.  
CCAImage 클래스는 현재의 snap 사진 위치를 결정하기 위한 몇개의 변수를 가지고 있는데,  
snap() 메소드 외부에서 setSnapRegion() 메소드를 호출할 수 있다면 카메라 기록시스템에 심각한 문제를 일으킬 수 있다.  
이미지 클래스를 제대로 테스트 할 수 없는 이유는 이 클래스가 너무 많은 책임을 갖고 있기 때문이다.  
이상적으로는 이 클래스를 작은 클래스로 나누는 것이 좋지만, 여러가지를 검토하여야 한다(시간, 위험요소 파악 등...)  
지금 당장 클래스의 책임을 분할할 수 없는 경우에도 테스트 루틴을 작성할 수 있을까? 다행히도 가능하다.  

우선 setSnapRegion을 protected로 변경한다.  
```Cpp
Class CCAImage
{
protected:
  void setSnapRegion(int x, int y, int dx, int dy);
  ...
public:
  void snap();
  ...
};
```
이어서 이 메소드에 접근하기 위한 서브클래스를 만든다.  

```Cpp
Class TestingCCAImage : public CCAImage
{
public:
  void setSnapRegion(int x, int y, int dx, int dy)
  {
    //상위 클래스의 setSnapRegion 호출
    CCAImage::setSnapRegion(x, y, dx, dy);
  }
};
```
이렇게 하면 테스트 루틴 내에서 CCAImage에 있는 setSnapRegion을 간접적으로 호출할 수 있다.  
하지만 이것이 좋은 아이디어일까? 메소드를 public으로 만들고 싶지 않아서 사용한 방법인데 결국 비슷한 측면이 있다.  
하지만 테스트를 가능하게 해준다는 점에 충분히 이점이 있다.  
물론 캡슐화를 위반한것은 사실이며, 코드의 동작에 대해 조사할 때는 setSnapRegion 메소드가 서브클래스 내에서 호출될 수 있음을 고려해야 하지만 이는 사소한 문제일 뿐이다.  
protected 메소드는 언젠가 이 클래스를 수정할 때 전체적인 리팩토링을 하기로 결정하게 만드는 계기가 될 수 있고,  
이때 CCAImage의 책임을 여러 클래스로 분리한 후 각각을 테스트 루틴으로 보호하면 된다.

## 언어의 편리한 기능
프로그래밍 언어의 설계자들은 개발자가 더 편해지게 하려고 최대한 노력하지만 프로그래밍의 단순함과 보안성 및 안정성 간에 균형을 유지해야 하기에 쉬운일이 아니다.  
다음 코드는 웹 클라이언트로부터 업로드된 파일들의 컬렉션을 얻어오는 C#프로그램의 일부분이다.  
이 코드는 컬렉션 내의 모든 파일을 순회하면서 특별한 특성을 갖는 파일과 관련된 스트림 목록을 반환한다.  

```C#
public void Ilist getKSRStreams(HttpFileCollection files){
  ArrayList list = new ArrayList();
  foreach(string name in files){
    HttpPostedFile file = files[name];
    if(file.FileName.EndsWith(".ksr") || (file.FileName.EndsWith(".txt") && file ContentLength > MIN_LEN)){
      ...
      list.Add(file.InputStream);
    }
  }
  return list;
}
```
이 코드에 변경을 가하면서 약간의 리팩토링을 하고 싶지만, 테스트 루틴 작성은 어려울것 같다.  
HttpFileCollection 객체를 생성하고 HttpPostedFile 객체를 전달하고 싶지만 이는 불가능한데,
 - HttpPostedFile 클래스는 public 생성자를 가지고 있지 않으며 이 클래스는 sealed 클래스이기 때문이다.(sealed클래스 : 서브클래스를 정의할 수 없는 클래스)
 - HttpPostedFile의 인스턴스를 생성할 수없을 뿐 아니라 서브클래스화도 불가능 하다.
 - 또한 HttpFileCollection 클래스 역시 동일한 문제를 갖고 있다.(sealed클래스는 아니지만 다른 문제가 동일하다)
 - 두 클래스는 라이브러리 클래스이므로 마음대로 제어할 수없으며 변경할 수도 없다.

따라서 유일하게 가능한 방법은 매개변수 적합 기법이다.  
다행히 코드에서 사용되는 sealed 클래스인 HttpFileCollection은 NameObjectCollectionBase라는 슈퍼클래스를 가지고 있다.  
따라서 이 클래스를 서브 클래스화 하고 그 서브클래스의 객체를 getKSRStreams 메소드에 전달 할 수 있다.  

```C#
public void LList getKSRStreams(OurHttpFileCollection files){
  ArrayList list = new ArrayList();
  foreach(string name in files){
    HttpPostedFile file = files[name];
    if(file.FileName.EndsWith(".ksr") || (file.FileName.EndsWith(".txt") && file ContentLength > MIN_LEN)){
      ...
      list.Add(file.InputStream);
    }
  }
  return list;
}
```
OurHttpFileCollection은 NameObjectCollectionBase의 서브클래스이며, NameObjectCollectionBase는 문자열을 객체와 연결하는 추상 클래스다.  
더 어려운 문제가 남아있다. 테스트 루틴에서 getKSRStreams를 실행하려면 HttpPostedFiles 객체가 필요한데 이 객체를 생성할 수 없기 때문이다.  
이 객체에서 우리가 실제로 필요한 것은 FileName과 ContentLength 이 두개의 속성을 제공하는 클래스가 필요한 것 같다.  
HttpPostedFile 클래스를 분리하기 위해 API 포장 기법을 사용할 수 있다.  
이를 위해 인터페이스(IHttpPostedFile)를 추출하고 래퍼(HttpPostedFileWrapper)를 작성해보자.  

```C#
public class HttpPostedFileWrapper : IHttpPostedFile
{
  public HttpPostedFileWrapper(HttpPostedFile file) {
    this.file = file;
  }
  public int ContentLength {
    get { return file.ContentLength; }
  }
  ...
}
```
인터페이스가 준비되었으니 테스트용의 가짜 클래스도 작성할 수 있다.  

```C#
public class FakeHttpPostedFile : IHttpPostedFile
{
  public FakeHttpPostedFile(int length, Stream stream, ...) { ... }
  public int ContentLength {
    get { retrun length; }
  }
}
```

```C#
public void IList getKSRStreams(OurHttpFileCollection){
  ArrayList list = new ArrayList();
  foreach(string name in files){
    IHttpPostedFile file = files[name];
    if(file.FileName.EndsWith(".ksr") || (file.FileName.EndsWith(".txt") && file ContentLength > MIN_LEN)){
      ...
      list.Add(file.InputStream);
    }
  }
  return list;
}
```
이 방법의 유일한 단점은 배포 코드에 원래의 HttpFileCollection 코드를 순회하면서 그 안에 포함된 각각의 HttpPostedFile을 포장한 후,  
그것을 getKSRStreams메소드에 전달되는 컬렉션에 추가해야 한다는 점이다.  
이는 보안을 위해 치러야 하는 불가피한 대가라고 할 수 있다.  
sealed와 final 클래스는 프로그래밍 언어에 존재하면 안되는 기능이 아니며, 사실 잘못은 개발자 쪽에 있다.  
제어 범위를 벗어난 라이브럴리에 직접 의존함으로서 스스로 어려움을 초래한 것이다.

## 탐지 불가능한 부작용
한개의 기능을 위해 테스트 루틴을 작성하는 것이 이론적으로 나쁘지는 않다.  
이 객체가 다른 객체와 아무것도 주고받지 않는다면 확실히 그렇지만, 문제는 다른 객체를 사용하지 않는 객체가 거의 없다는 점이다.  
가끔 값을 반환하지 않는 객체들이 있는데, 이런 메소드를 호출하면 어떠한 작업이 수행되겠지만 우리(호출코드)는 구체적인 작업 내용을 절대 알 수 없다.  
다음은 이런 문제를 가진 클래스의 예이다.
```JAVA
public class AccountDetailFrame extends Frame implements ActionListener, WindowListener{
  private TextField display = new TextField(10);
  ...
  public AccountDetailFrame(...) { ... }
  
  public void actionPerformed(ActionEvent event) {
    String source = (String)event.getActionCommand();
    if(source.equals("project activity")) {
      detailDisplay = new DetailFrame();
      detailDisplay.setDescription(getDetailText() + " " + getProjectionText());
      detailDisplay.show();
      String accountDescription = detailDisplay.getAccountSymbol();
      accountDescription += ": ";
      ...
      display.setText(accountDescription);
      ...
    }
  }
  ...
}
```

이 자바 클래스는 다양한 일을한다.  
 1. GUI 컴포넌트들을 생성하고
 2. actionPerformed 핸들러를 통해 결과를 수신하며
 3. 화면에 표시될 값을 계산한 후 그 결과를 표시한다.
이 모든 작업을 매우 특이한 방식으로 수행한다. 상세 텍스트를 생성한 후, 다른 윈도우를 생성하여 화면에 표시한다.  
테스트 하네스 네에서 이 메소드의 실행을 시도할 수 있지만 별 의미가 없다.  
윈도우를 생성하고 이를 화면에 띄우며 입력값을 요구하는 프롬프트를 보여주고 다른 윈도우에 무언가를 표시한다.  
그중에서 코드가 무슨일을 하는지 탐지할 수 있는 부분은 없다.  
그럼 어떻게 해야할까? 우선 GUI로부터 독립적인 부분과 GUI에 의존적인 부분을 물리적으로 분리하는 것부터 시작하자.  
첫번째 단계로 메소드 추출기법을 통해 리팩토링을 수행함으로서 메소드의 처리 내용을 분리해야 한다.  
이 메소드 자체는 윈도우 프래임워크로부터 받은 통지에 대한 훅 함수인데, 가장먼저 할일은 전달받은 ActionEvent로부터 명령의 이름을 얻는것이다.  
메소드 본문 전체를 추출하면 ActionEvent 클래스에 대한 의존을 전부 분리할 수 있다.  

```JAVA
public class AccountDetailFrame extends Frame implements ActionListener, WindowListener{
  private TextField display = new TextField(10);
  ...
  public AccountDetailFrame(...) { ... }
  
  public void actionPerformed(ActionEvent event) {
    String source = (String)event.getActionCommand();
    performCommand(source);
  }
  
  public void performCommand(String source) {
    if(source.equals("project activity")) {
      detailDisplay = new DetailFrame();
      detailDisplay.setDescription(getDetailText() + " " + getProjectionText());
      detailDisplay.show();
      String accountDescription = detailDisplay.getAccountSymbol();
      accountDescription += ": ";
      ...
      display.setText(accountDescription);
      ...
    }
  }
  ...
}
```
하지만 아직 테스트 가능한 코드로서는 충분하지 않다. 그다음 할 일은 다른 프레임에 접근하는 코드 부분을 메소드로 추출하는 것이다.  
이를 위해서는 detailDisplay변수를 이 클래스의 인스턴스 변수로 만드는 것이 좋다.

```JAVA
public class AccountDetailFrame extends Frame implements ActionListener, WindowListener{
  private TextField display = new TextField(10);
  private DetailFrame detailDisplay;
  ...
  public AccountDetailFrame(...) { ... }
  
  public void actionPerformed(ActionEvent event) {
    String source = (String)event.getActionCommand();
    performCommand(source);
  }
  
  public void performCommand(String source) {
    if(source.equals("project activity")) {
      detailDisplay = new DetailFrame();
      detailDisplay.setDescription(getDetailText() + " " + getProjectionText());
      detailDisplay.show();
      String accountDescription = detailDisplay.getAccountSymbol();
      accountDescription += ": ";
      ...
      display.setText(accountDescription);
      ...
    }
  }
  ...
}
```
이제 프레임을 사용하는 코드를 일련의 메소드들로 추출할 수 있게 됐다.  
다음 코드는 몇번의 추출이 있은 후에 perforrmCommand 메소드의 모습을 보여준다.  

```JAVA
public class AccountDetailFrame extends Frame implements ActionListener, WindowListener{
  ...  
  public void performCommand(String source) {
    if(source.equals("project activity")) {
      setDescription(getDetailText() + " " + getProjectionText());
      ...
      String accountDescription = getAccountSymbol();
      accountDescription += ": ";
      ...
      display.setText(accountDescription);
      ...
    }
  }
  
  void setDescription(String description) {
    detailDisplay = new DetailFrame();
    detailDisplay.setDescription(description);
    detailDisplay.show();
  }
  
  String getAccountSymbol() {
    return detailDisplay.getAccountSymbol();
  }
  
  ...
}
```
detailDisplay 프레임과 연관된 모든 코드를 추출했으니 이제 AccountDetailFrame의 컴포넌트에 접근하는 코드를 추출할 수 있다.  

```JAVA
public class AccountDetailFrame extends Frame implements ActionListener, WindowListener{
  ...  
  public void performCommand(String source) {
    if(source.equals("project activity")) {
      setDescription(getDetailText() + " " + getProjectionText());
      ...
      String accountDescription = getAccountSymbol();
      accountDescription += ": ";
      ...
      setDisplayText(accountDescription);      
      ...
    }
  }
  
  void setDescription(String description) {
    detailDisplay = new DetailFrame();
    detailDisplay.setDescription(description);
    detailDisplay.show();
  }
  
  String getAccountSymbol() {
    return detailDisplay.getAccountSymbol();
  }
  
  void setDisplayText(String description) {
    display.setText(accountDescription);
  }
  
  ...
}
```
이러한 추출이 모두 끝나고 나서야 비로소 서브클래스화와 메소드 재정의 기법을 적용해 performCommand 메소드 내에 남아있는 코드를 테스트할 수 있다.  
예를들어 AccountDetailFrame을 다음과 같이 서브클래스화 하면 project activity 명령이 입력됐을 때 화면에 적절한 텍스트가 표시되는지 검증할 수 있다.  

```JAVA
public class TestingAccountDetailFrame enxtends AccountDetailFrame {
  String displayText = "";
  String accountSymbol = "";
  
  void setDescription(String description) {    
  }
  
  String getAccountSymbol() {
    return accountSymbol;
  }
  
  void setDisplayText(String text) {
    displayText = text;
  }
}
```
다음 코드는 performCommand 메소드를 실행하는 테스트 코드다.

```JAVA
public void testPerformCommand() {
  TestingAccountDetailFrame frame = new TestingAccountDetailFrame();
  frame.accountSymbol = "SYM";
  frame.performCommand("project activity");
  assertEquals("SYM: basic account", frame.displayText);
}
``` 
