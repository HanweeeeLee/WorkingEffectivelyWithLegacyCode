# 08. 어떻게 기능을 추가할까?
적절한 위치에 테스트 루틴이 존재하게 되면 기능을 추가하기 편리한 상황이 된다. 즉 테스트 루틴은 단단한 기반이 된다.

## 테스트 주도 개발
테스트 주도 개발 이란 문제를 해결할 수 있는 메소드를 상상하고 이에 대해 실패하는 테스트 케이스를 작성하는 것이다.  
문제를 해결할 수 있는 메소드가 아직 완성되지 않았지만 테스트 루틴을 먼저 작성함으로서 앞으로 작성할 코드가 무엇을 구현할지 명쾌히 이해할 수 있게 된다.
테스트 주도 개발은 다음과 같이 간단한 알고리즘을 사용한다.  
1. 실패하는 테스트 케이스를 작성한다.
2. 컴파일한다.
3. 테스트를 통과한다.
4. 중복을 제거한다.
5. 반복한다.

다음은 금융관련 애플리케이션의 작성 예제이다.

### 실패 테스트 케이스 작성
지금 필요로 하는 기능을 위한 테스트 케이스는 다음과 같다.  
금융 관련 애플리케이션의 예제인데 어떤 상품이 거래되야 할지 검증하는 수학계산을 수행하는 클래스가 필요하다.  
이를 위해 1차 적률이라는것을 계산하는 자바클래스를 작성해야하는데, 계산을 수행하는 메소드는 없지만 메소드를 테스트할 수 있는 테스트 케이스는 작성할 수 있다.  
테스트에 사용되는 값을 사용한 계산 결과가 -0.5가 되야 한다는 것은 미리 알고 있기 때문이다.  
```Java
public void testFirstMoment() {
    InstrumentCalculator calculator = new InstrumentCalculator();
    calculator.addElement(1.0);
    calculator.addElement(2.0);
    
    assertEquals(-0.5, calculator.firstMomentAbout(2.0), TOLERANCE);
}
```

### 컴파일
위의 테스트 루틴은 컴파일 되지는 않는다. InstrumentCalculator 클래스 안에 firstMomentAbout라는 이름의 메소드가 없기 때문이다.  
테스트가 실패하도록 이 메소드를 공백 메소드로서 추가해준다. 결과값으로 -0.5가 아니라 NaN이 반환되게 한다.  
```Java
public class InstrumentCalculator {
    double firstMomentAbout(double point) {
      return Double.NaN;
    }
    ...
}
```

### 테스트 통과시키기
테스트를 실행할 수 있으니, 이제 테스트를 통과하는 코드를 작성한다.
```Java
public double firstMomentAbout(double point) {
    double numerator = 0.0;
    
    for(Iterator it = elements.iterator(); it.hasNext(); ) {
      double element = ((Double)(it.next())).doubleValue();
      numerator += element - point;
    }
    
    return numerator / elements.size();
}
```

### 중복제거
현재로서는 중복부분이 없으므로 pass

### 실패 테스트 케이스 작성
앞의 코드는 테스트를 통과하겠지만 모든 테스트 케이스를 통과하는 것은 아니다.
반환문에서 의도치 않게 0으로 나누기를 수행할 수 있다. 만일 인수로서 전달받은 element가 하나도 없을 떄는 예외가 발생되야 할 것이다.  
다음과 같이 수정 하자.
```Java
public void testFirstMoment() {
    try {
        new InstrumentCalculator().firstMomentAbout(0.0);
        fail("expected InvalidBasisException");
    } catch(InvalidBasisException e) {
    }
}
```

### 컴파일
InvalidBasisException 예외를 발생시키도록 firstMomentAbout 선언을 수정해야 한다.
```Java
public double firstMomentAbout(double point) throws InvalidBasisException {
    double numerator = 0.0;
    
    for(Iterator it = elements.iterator(); it.hasNext(); ) {
      double element = ((Double)(it.next())).doubleValue();
      numerator += element - point;
    }
    
    return numerator / elements.size();
}
```
하지만 이렇게 해도 컴파일되지는 않는다. 컴파일러는 firstMomentAbout 예외를 실제로 발생시켜야 한다고 알릴 것이다 따라서 다음을 추가한다.
```Java
public double firstMomentAbout(double point) throws InvalidBasisException {
    if (element.size() == 0)
      throws new InvalidBasisException("no Elements");

    double numerator = 0.0;
    
    for(Iterator it = elements.iterator(); it.hasNext(); ) {
      double element = ((Double)(it.next())).doubleValue();
      numerator += element - point;
    }
    
    return numerator / elements.size();
}
```

### 테스트 통과시키기
이제 문제없이 테스트를 통과한다.

### 중복제거
이번에도 중복부분이 없으므로 pass

### 테스트 실패 케이스 작성
이제 2차 적률을 계산하는 메소드를 작성해야 한다. 2차 적률은 확률 변수의 분산값을 의미하며 예상값은 0.5 이다.  
아직 존재하지 않는 메소드 secondMomentAbout에 대한 테스트 루틴을 작성한다.  
```Java
public void testSecondMoment() {
    InstrumentCalculator calculator = new InstrumentCalculator();
    calculator.addElement(1.0);
    calculator.addElement(2.0);
    
    assertEquals(0.5, calculator.secondMomentAbout(2.0), TOLERANCE);
}
```
### 컴파일
컴파일을 위해서는 secondMomentAbout의 정의를 추가해야한다. firstMomentAbout에서 사용한 방법을 그대로 적용하자.  
컴파일 되도록 firstMomentAbout을 복사하여 secondMomentAbout로 이름을 바꿔주자  
```Java
public double secondMomentAbout(double point) throws InvalidBasisException {
    if (element.size() == 0)
      throws new InvalidBasisException("no Elements");

    double numerator = 0.0;
    
    for(Iterator it = elements.iterator(); it.hasNext(); ) {
      double element = ((Double)(it.next())).doubleValue();
      numerator += element - point;
    }
    
    return numerator / elements.size();
}
```

### 테스트 통과시키기
좀전의 코드는 테스트에 실패한다. 다음과 같이 수정해주자.  

기존의 firstMomentAbout의 다음 부분을
```Java
numerator += element - point;
```
2차 적률 계산식인 다음과 같이 변경하자
```Java
numerator += Math.pow(element - point, 2.0);
```
이 계산식은 다음과 같은 n차 적률을 계산하는 일반적인 수식이다
```Java
numerator += Math.pow(element - point, N);
```
numerator += element - point는 numerator += Math.pow(element - point, 1.0)과 같은 값을 가지기에 firstMomentAbout의 코드는 제대로 작동한다.  

```Java
public double secondMomentAbout(double point) throws InvalidBasisException {
    if (element.size() == 0)
      throws new InvalidBasisException("no Elements");

    double numerator = 0.0;
    
    for(Iterator it = elements.iterator(); it.hasNext(); ) {
      double element = ((Double)(it.next())).doubleValue();
      numerator += Math.pow(element - point, 2.0);
    }
    
    return numerator / elements.size();
}
```

### 중복 제거
두 개의 테스트 firstMomentAbout와 secondMomentAbout를 통과 했으니 다음 단계인 중복 제거를 진행해 보자.  
어떻게 중복을 제거할수 있을까?  
secondMomentAbout의 본문 전체를 추출해서 nthMomentAbout를 작성한 후 매개변수 N을 전달하는 방법이 있다.  
```Java
public double secondMomentAbout(double point) throws InvalidBasisException {
    return nthMomentAbout(point, 2.0);
}

private double nthMomentAbout(double point, double n) throws InvalidBasisException {
   if (element.size() == 0)
      throws new InvalidBasisException("no Elements");

    double numerator = 0.0;
    
    for(Iterator it = elements.iterator(); it.hasNext(); ) {
      double element = ((Double)(it.next())).doubleValue();
      numerator += Math.pow(element - point, n);
    }
    
    return numerator / elements.size();
}
```
테스트는 문제없이 통과한다 firstMomentAbout에서도 nthMomentAbout를 호출하도록 변경할 수 있다.
```Java
public double firstMomentAbout(double point) throws InvalidBasisException {
    return nthMomentAbout(point, 1.0);
}
```
레거시 코드에서 TDD 알고리즘은 다음과 같이 확장될 수 있다.
0. 변경대상 클래스를 테스트루틴으로 보호한다.
1. 실패 테스트 케이스를 작성한다.
2. 컴파일
3. 테스트 통과시키기(이때 가능한 기존 코드를 변경하지 않는다.)
4. 중복 제거
5. 반복

## 차이에 의한 프로그래밍

