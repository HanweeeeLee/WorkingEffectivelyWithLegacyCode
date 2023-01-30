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



## 전방 추론

## 영향 전파

## 영향 분석을 통한 학습

## 영향 스케치의 단순화
