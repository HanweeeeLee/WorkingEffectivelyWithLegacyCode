# 11. 코드를 변경해야 한다. 어느 메소드를 테스트해야 할까?
코드를 변경해야 하는데, 기존 동작을 명확히 하기 위해 문서화 테스트를 작성해야 한다고 하자. 이 테스트를 어디에 작성해야 할까?  
가장 간단한 답은 변경 대상 메소드마다 테스트 코드를 작성하는 것이다.  
하지만 코드가 간단하고 이해하기 쉽다면 충분하지만 레거시 코드는 어떤 위치의 코드를 변경했을때 다른 위치의 동작이 변경될 수 있다.  
따라서 테스트 루틴이 적절히 위치하지 않으면 어떤 영향을 미칠지 알 길이 없다.  
복잡한 코드를 이해하기 쉽도록 리팩토링 해야 함은 알고 있지만, 여기서 다시 테스트가 문제가 된다.  
테스트 코드가 존재하지 않는다면 리팩토링을 정확히 하고 있는지 어떻게 알수 있을까?  
이번장은 이런 간극을 좁히기 위한 기법들을 설명한다.  

## 영향 추론
```C#
int getBalancePoint() {
  const int SCALE_FACTOR = 3;
  int result = startingLoad + (LOAD_FACTOR * residual * SCALE_FACTOR);
  foreach(Load load in loads){
    result += load.getPointWeight() * SCALE_FACTOR;
  }
  return result;
}
```
다음의 C# 코드에서 3을 4로 변경하면, 이 메소드를 호출했을 때의 반환 값이 달라진다.  
이는 이 메소드를 호출한 다른 메소드의 결과 값도 바꾸며, 연쇄적으로 이어지면 결국 시스템의 경계까지 영향을 미칠수 있다.  
이런 영향에도 불구하고 코드의 대부분은 기존 동작이 달라지지 않는다. 대부분의 코드는 getBalancePoint()를 직접적이든 간접적이든 호출하지 않기 떄문에 결과도 달라지지 않기 때문이다.  

다음의 자바 클래스는 C++ 소스코드를 해석하는 애플리케이션의 일부다.  
다음의 CppClass 객체를 생성한 후에 수행 가능한 변경 중에서, CppClass 객체 내 메소드들의 반환 값에 영향을 미칠 수 있는 것의 목록을 만들어보자  

```JAVA
public class CppClass {
  private String name;
  private List declarations;
  
  public CppClass(String name, List declarations){
    this.name = name;
    this.declarations = declarations;
  }
  
  public int getDeclarationCount() {
    return declarations.size();
  }
  
  public String getName() {
    return name;
  }
  
  public Declaration getDeclaration(int index) {
    return ((Declaration)declartions.get(index));
  }
  
  public String getInterface(String interfaceName, int[] indices) {
    String result = "class " + interfaceName + " {\npublic:\n";
    for(int n=0;n<indices.length;n++) {
      Declaration virtualFunction = (Declaration)(declarations.get(indices[n]));
      result += "\t" + virtualFunction.asAbstract() + "\n";
    }
    result += "};\n";
    return result;
  }  
}
```
이 목록은 다음과 같다.  
 1. 생성자에게 declarations 인수로 선언문 리스트를 전달한 후, 나중에 신규 요소가 이 리스트에 추가될 수 있다. 이 리스트는 참조에 의해 전달되므로 getInterface, getDeclaration, getDeclartationCount 메소드의 결과 값이 바뀌는 영향을 미칠 수 있다.
 2. 선언문 리스트 내의 객체중 하나가 변경 되거나 대체되면 역시 좀 전에 언급한 메소드들에 영향을 미칠 수 있다.

다음은 각 메소드에 영향을 미치는 다이어그램 들이다.
![KakaoTalk_20230130_185105968](https://user-images.githubusercontent.com/50142323/215444352-db6feba8-89f9-4911-8477-3d1d9a08e1e8.jpg)
![KakaoTalk_20230130_185105968_01](https://user-images.githubusercontent.com/50142323/215444380-8eec59c2-af68-49be-b7ec-24ce2ee7813f.jpg)
![KakaoTalk_20230130_185105968_02](https://user-images.githubusercontent.com/50142323/215444405-b6df7abd-e195-45b9-85ae-471b38925fa7.jpg)

이 다이어그램을 그리는 규칙은 간단하다.  
핵심은 영향을 받는 변수와 반환값이 바뀔 수 있는 메소드를 타원으로 나타내는 것이다.  
변수는 동일 객체 내에 있을수도, 다른 객체 내에 있을수도 있지만 어느쪽이든 중요치 않다.  
값이 바뀔 변수를 타원으로 그린 후, 변수 값의 변경으로 인해 반환 값이 바뀔 수 있는 메소드를 향해 화살표를 그린다.  

품질이 떨어지는 프로그램에서는 결과값이 왜 이렇게 나오는지 파악하기 매우 어려울 때가 많다.  
레거시코드를 다룰 때는 수시로 다음 질문에도 대답해야 한다. "어떤 변경을 수행하면, 그 변경이 프로그램의 나머지 부분에 어떤 영향을 미칠까?"  
이 질문에 답하려면 변경 지점을 기준으로 전방 추론을 해야한다.  
전방 추론을 능숙하게 하는 것은 곧 테스트 루틴의 최적 위치를 찾는 기법의 기초를 닦는것이다.

## 전방 추론
앞의 예제에서는 코드의 특정 위치에 있는 값에 대해 어떤 객체들이 영향을 미치는지 추론했다.  
하지만 문서화 테스트를 작성할 때는 이와 반대로 추론한다. 즉 먼저 객체들을 조사하고 이 객체들이 제대로 동작하지 않을 경우 어떤 영향을 미치게 되는지 추론한다.  
다음 클래스를 보면 인메모리 파일 시스템의 일부이다. 이 클래스에 대한 테스트 루틴이 존재하지 않지만 이 클래스에 대한 변경을 수행해야 하는 상황이다.  

```JAVA
public class InMemoryDirectory {
  private List elements = new ArrayList();
  public void addElement(Element newElement) {
    elements.add(newElement);
  }
  
  public void generateIndex() {
    Element index = new Element("index");
    for(Iterator it = elements.iterator(); it.hasNext();) {
      Element current = (Element)it.next();
      index.addText(current.getName() + "\n");
    }
    addElement(index);
  }
  
  public int getElementCount() {
    return elements.size();
  }
  
  public Element getElement(String name) {
    for(Iterator it = elements.iterator(); it.hasNext();) {
      Element current = (Element)it.next();
      if(current.getName().equals(name)) {
        return current;
      }
    }
    return null;
  }  
}
```
InMemoryDirectory는 자바 클래스며, InMemoryDirectory 객체를 생성한 후 요소를 추가하고 인덱스를 생성해 접근할 수 있다.  
여기서 Element는 파일과 마찬가지로 텍스트를 포함하는 객체다. 요소에대한 인덱스를 생성할 때는 index라는 이름의 Element객체를 생성하고 index의 텍스트에 다른 모든 요소들의 이름을 추가한다.  
InMemoryDirectory의 특이한 점 중 하나는 generateIndex를 두번 호출하면 인덱스가 엉망이 된다는 점이다.  
generateIndex를 두번 호출하면 두개의 인덱스가 생기게 되며, 나중에 생성된 인덱스는 처음 생성된 인덱스를 요소로서 포함하게 된다.  
다행히 애플리케이션 내에서 InMemoryDirectory는 매우 제한적으로만 사용되고 현재 잘 돌아가고 있지만 이를 변경할 필요가 생겼다고 하자.  
디렉터리가 존재하는 동안에는 언제든지 요소 추가가 가능하도록 소프트웨어를 변경해야 한다.  
이상적으로는 요소추가와 동시에 인덱스를 생성하고 유지보수 하는것이 바람직하다.  
신규 동작을 위해 테스트 루틴을 작성하는 것과 테스트를 만족시키기 위해 코드를 작성하는 것은 매우 간단하지만, 현재의 동작에 대한 테스트 루틴은 어디에도 없다.  
이 테스트 루틴을 어느곳에 작성해야 할지 어떻게 알 수 있을까?  

이번 예제의 경우에는 다양한 방식으로 addElement를 호출하고 인덱스를 생성하며 요소들이 제대로 저장됐는지 확인하는 일련의 테스트 루틴이 필요하다.  
하지만 테스트 루틴의 위치를 결정하는 문제는 간단하지 않다.  
이 예제에서 가장 먼저 할일은 변경해야 할 위치를 판단하는 것이다. 여기서는 generateIndex 메소드에서 기능을 제거하고 addElement에 기능을 추가해야 한다.  
먼저 generateIndex 메소드를 무엇이 호출하는가? 클래스 내의 다른 메소드는 호출하지 않고 오직 다른 클래스에 의해서만 호출될 뿐이다.  
generateIndex에 대해 무엇을 변경하려는 것일까? 신규 요소를 생성하고 디렉터리에 이 요소를 추가할 것이므로 generateIndex는 클래스의 elements 변수가 참조하는 컬렉션에 영향을 미친다.  
이제 elements 컬렉션에 주목하자. 이 컬렉션은 어느곳에서 사용될까? getElementCount와 getElement메소드에서 사용되는것 같다.  
elements 컬렉션은 addElement에서도 사용되지만 addElement 메소드는 elements 컬렉션의 상태와 관계없이 동작하기 때문에대상에서 제외할 수 있다.  
elements 컬렉션에 무슨일을 하든 addElement를 호출한 코드는 영향을 받지 않는다.  
![KakaoTalk_20230130_194551383](https://user-images.githubusercontent.com/50142323/215456200-2e26bff7-a5f8-493c-ba95-e7cf598bcfce.jpg)

변경지점이 generateIndex와 addElement 메소드 이므로 addElement 메소드가 주변 소프트웨어에 어떻게 영향을 미치는지도 살펴봐야 한다.  
addElement는 elements 컬렉션에 영향을 미치는것 같다.  
![KakaoTalk_20230130_194825885](https://user-images.githubusercontent.com/50142323/215456781-8b6edf87-6792-4c12-9d66-130224625c08.jpg)

전체 영향 스케치는 다음과 같다.  
![KakaoTalk_20230130_194825885_01](https://user-images.githubusercontent.com/50142323/215456887-0dd6fdeb-a5b3-420e-9554-748365c3ea5e.jpg)

따라서 InMemoryDirectory 클래스의 사용자가 이러한 변경에 의한 영향을 감지하려면 getElementCount와 getElement 메소드를 통해 확인해야 한다.  
이 두개에 대한 테스트 루틴을 작성할 수 있다면, 변경에 의한 영향을 모두 아우를 수 있을것 같다.  
혹시 뭔가 빠뜨린게 있지는 않을까? 만약 InMemoryDirectory 내에 public, protected, package로 선언된 데이터가 있을 경우, 서브클래스 혹은 동일 패키지 내의 다른 메소드에서 이 데이터를 변경할 수 있다.  
하지만 InMemoryDirectory의 인스턴스 변수는 private로 선언 됐으므로 그런 걱정을 할 필요 없다.  
이제 끝난걸까? 지금까지 완전히 놓친것이 하나 있다.  
generateIndex를 호출하면 Element 객체가 생성되고 이 객체의 addText 메소드가 반복 호출된다.  
Element 클래스의 코드를 살펴보자.  

```JAVA
public class Element {
  private String name;
  private String text = "";
  public Element(String name) {
    this.name = name;
  }
  
  public String getName() {
    return name;
  }
  
  public void addText(String newText) {
    text += newText;
  }
  
  public String getText() {
    return text;
  }
}
```
다행히 코드가 간단하다. generateIndex가 생성하는 신규요소를 그림으로 나타내보자.  
![KakaoTalk_20230130_200544285](https://user-images.githubusercontent.com/50142323/215460596-22f69850-899b-4af6-9156-569931e3aa4e.jpg)

텍스트가 있는 하나의 새로운 요소를 가질 때, generateIndex는 그것을 컬렉션에 추가하기 때문에 새로운 요소는 컬렉션에 영향을 미친다.  
![KakaoTalk_20230130_200544285_01](https://user-images.githubusercontent.com/50142323/215460803-859fdaae-c100-462f-9bea-8a7866fe43ab.jpg)

지금까지의 작업을 통해 addText 메소드가 elements 컬렉션에 영향을 미치고, 이후 getElement와 getElementCount의 반환값에 영향을 준다는 것을 확인했다.  
따라서 getElement와 getElementCount 메소드는 변경 영향을 감지하는 테스트 루틴을 작성하기 위한 최적의 위치이다.  

이번예제는 크기는 작지만 레거시코드에서 변경 영향을 평가할 때 어떤 식으로 추론해야 하는지 잘 보여준다.  
테스트위치를 찾아야할때 가장 먼저 할 일은 어디서 변경을 탐지할 수 있는지, 즉 변경의 영향이 무엇인지 파악하는 것이다.  
어디서 영향을 탐지할 수 있는지 알아내면, 그중 하나를 골라서 그곳에 테스트 루틴을 작성하면 된다.

## 영향 전파
영향이 전파되는 과정을 찾기는 그다지 어렵지 않다.  
앞선 InMemoryDirectory 클래스의 경우 호출 코드에게 값을 반환하는 메소드들을 찾을 수 있었다. 이떄 변경 지점에서 영향 추적을 시작했지만, 일반적으로는 반환 값을 갖는 메소드를 먼저 주목하는 경우가 많다.  
반환 값이 무시되지 않는 한, 반환 값은 호출 코드에 영향을 미치기 때문이다.  
또는 다른 객체를 매개변수로서 받는 객체는 그 객체의 상태를 변경할 수 있으며 이 변경이 조용하게 애플리케이션의 나머지 부분에 영향을 미치게 되기도 한다.  
가장 은밀한 전파는 전역변수 또는 정적변수에 의해 일어난다. 다음 예제를 보자.

```JAVA
public class Element {
  private String name;
  private String text = "";
  public Element(String name) {
    this.name = name;
  }
  
  public String getName() {
    return name;
  }
  
  public void addText(String newText) {
    text += newText;
    View.getCurrentDisplay().addText(newText);
  }
  
  public String getText() {
    return text;
  }
}
```
위 Element 클래스 내 메소드의 서명만 봐서는 Element가 View 객체에 영향을 미치는지 알 수 없다.  
정보 은닉이 필요할 수는 있지만, 알 필요가 없는 정보에만 적용하는것이 바람직하다.  

다음은 필자가 경험한 변경 영향을 탐색할 때 사용하는 방법이다.  
 1. 변경 대상 메소드를 식별한다.
 2. 메소드가 반환값을 가진다면, 이 메소드를 호출하는 코드를 살펴본다.
 3. 메소드가 어떤 값을 변경하는지 여부를 살펴본다. 변경하는 값이 있다면, 그 값을 사용하는 메소드와 다시 이 메소드를 사용하는 메소드를 살펴본다.
 4. 인스턴스 변수 및 메소드 사용자가 될 수 있는 슈퍼클래스와 서브클래스도 잊지말고 찾아본다.
 5. 메소드에 전달되는 매게변수를 살펴본다. 변경하고자 하는 코드가 매개변수나 반환값인 객체를 변경하지 않는지 살펴본다.
 6. 식별된 메소드 중에서 전역 변수나 정적 변수를 변경하는 것이 있는지 살펴본다.

## 영향 추론을 위한 도구
우리가 갖고 있는 무기 중에서 가장 중요한 것은 바로 프로그래밍 언어에 대한 지식이다.  
어떤 언어든 영향 전파를 막기 위한 방화벽이 존재한다.  
그 방화벽이 무엇인지 알고 있다면, 굳이 사용하지 않을 이유는 없다.  

다음과 같은 Coordinate 클래스의 내부 구현을 변경하고 싶다고 하자.  
3차원 및 4차원 좌표를 나타낼 수 있도록 Coordinate 클래스를 일반화 하고, 벡터를 사용해 x값과 y값을 유지하도록 변경하려고 한다.  
두개의 코드를 비교해 보자

```JAVA
public class Coordinate {
  private double x = 0;
  private double y = 0;
  
  public Coordinate() {}
  public Coordinate(double x, double y) {
    this.x = x;
    this.y = y;
  }
  
  public double distance(Coordinate other) {
    return Math.sqrt(Math.pow(other.x - x, 2.0) + Math.pow(other.y - y, 2.0));
  }
}
```

```JAVA
public class Coordinate {
  double x = 0;
  double y = 0;
  
  public Coordinate() {}
  public Coordinate(double x, double y) {
    this.x = x;
    this.y = y;
  }
  
  public double distance(Coordinate other) {
    return Math.sqrt(Math.pow(other.x - x, 2.0) + Math.pow(other.y - y, 2.0));
  }
}
```
무슨 차이가 있을까? 첫번째 코드는 변수 x와 y가 private로 선언되었고, 두번째 코드는 package로 선언됐다.  
첫번째 코드에서 변수 x와 y의 값을 어떤식으로 변경하든, Coordinate 클래스를 호출하는 코드에는 distance 함수를 통해서만 영향을 미칠 수 있지만,  
두번째 코드에서는 동일 패키지 내의 코드는 변수 x와 y에 직접 접근할 수 있다.  
따라서 직접 접근하는 코드를 찾거나, 직접 접근하지 못하도록 private로 바꿔야 한다.  
Coordinate의 서브클래스 역시 인스턴스 변수에 접근할 수 있으므로, 인스턴스 변수들이 서브클래스 내의 메소드에서 사용되고 있는지도 확인해야 한다.  

언어의 미묘한 규칙이 원인이 돼서 오류를 범할 가능성이 있으므로, 사용 중인 언어를 정확히 아는 것이 중요하다.

## 영향 분석을 통한 학습
기회가 있을 때마다 영향을 분석해보는 것이 좋다. 코드 베이스에 익숙해지면 어떤 것은 찾지 않아도 된다고 느낄 수 있다.  
이런 느낌이 든다면, 코드베이스의 규칙들에 익숙해 졌다는 뜻이며 발생 가능한 영향을 찾느라 수고하지 않아도 된다.  
이러한 규칙을 발견하는 최상의 방법은 소프트웨어의 한 부분이 우리가 한번도 보지 못한 방식으로 다른 부분에 영향을 미칠 수 있음을 발견하는 것이다.
그러면 '이런 바보 같은'이라는 말이 나도 모르게 나오게 된다(ㄹㅇ..)

## 영향 스케치의 단순화
필자는 영향 스케치를 사용하면 매우 유용한것이 눈에 들어온다는 사실을 알려주고 싶었다고 한다.  
테스트 루틴을 작성할 위치를 찾을 때는 변경에 의해 무엇이 영향을 받을지 아는것이 중요하다.  
영향에 대해 추론하지 않으면 안된다. 약식으로 추론할 수도 있고 영향 스케치를 사용해 엄밀하게 추론할 수도 있다.  
하지만 가급적 영향 스케치를 배워두면 좋은점은, 매우 복잡한 코드를 대상으로 작업할 때, 테스트 루틴의 위치를 발견하는 과정에서 우리가 의지할 만한 몇 안 되는 기법이기 때문이다.

