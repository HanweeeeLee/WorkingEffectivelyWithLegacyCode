# 05. 도구

## 리팩토링 자동화 도구
> 리팩토링: 소프트웨어 기존 동작을 변경하지 않으면서, 이해 및 변경이 용이하도록 소프트웨어의 내부 구조를 변경하는 작업  

코드 변경이 리팩토링으로 인정 받으려면 기존 동작이 달라지지 않아야 한다. 따라서 리팩토링 도구는 코드 변겨이 동작 변경으로 이어지지 않음을 검증해야 하며, 실제로 많은 동구들이 이러한 검증 작업을 수행한다.  
리팩토링 도구는 신중하게 선택해야 하며, 도구 개발자가 도구의 안전성에 대해 어떻게 말하고 있는지 자세히 봐야 한다.  

### 테스트와 리팩토링 자동화
리팩토링 도구가 있으면 리팩토링 대상 코드를 위한 테스트 루틴을 작성할 필요가 없을까?
```Java
public class A {
    private int alpha = 0;
    private int getValue() {
        alpha++;
        return 12;
    }
    public void doSomething() {
        int v = getValue();
        int total = 0;
        for (int n = 0 ; n < 10 ; n++) {
            total += v;
        }
    }
}
```
적어도 두 개의 자바 리팩토링 도구에서 doSomething 메소드로부터 v라는 변수를 제거하는 자동 리팩토링을 수행할 수 있다. 
```Java
public class A {
    private int alpha = 0;
    private int getValue() {
        alpha++;
        return 12;
    }
    public void doSomething() {
        // int v = getValue();
        int total = 0;
        for (int n = 0 ; n < 10 ; n++) {
            // total += v;
            total += getValue();
        }
    }
}
```
> 변수 v는 제거됐지만, alpha 변수의 값이 한 번이 아니라 열 번 증가한다. 따라서 기존 동작이 변경되버렸다. 즉, 테스트 루틴 작성해야함

## 모조 객체
레거시 코드를 다룰 떄 가장 큰 문제점은 의존관계이다. 다른 코드를 제거한 상태로 특정 코드를 제대로 테스트하려면 그 다른 코드를 대신해서 올바른 값을 제공하는 또 다른 코드가 필요하다. 객체 지향에서는 이것을 일반적으로 모조 객체(mock object)라고 부른다.

## 단위 테스트 하네스
### xUnit 테스트
 - 좋음
 - 프로그래머는 현재 사용 중인 개발 언어로 테스트 루틴을 작성할 수 있다. 
 - 모든 테스트는 독립적으로 실행된다.
 - 테스트들을 그룹 단위로 묶어서 필요할 때마다 실행(또는 재실행)할 수 있다.

 ### JUnit
 JUnit에서는 TestCase라는 클래스를 상속받아 테스트 루틴을 작성할 수 있다.
 ```Java
 import junit.framework.*;

 public class FormulaTest extends TestCase {
    public void testEmpty() {
        assertEquals(0, new Formula("").value());
    }

    public void testDigit() {
        assertEquals(1, new Formula("1").value());
    }
 }

 ```
  - 테스트를 정의하려면, 테스트 클래스에 대해 테스트 케이스마다 void testXXX()라느 서명을 갖는 메소드를 작성한다. 
  - 테스트 메소드는 코드 및 assertXXX 메소드를 사용하는 확중문(assertion)을 포함할 수 있다.

#### 각각의 test메소드들이 각자의 객체를 만듦. 서로 영향을 주고받지 않는다.
```Java
public class EmployeeTest extends TestCase {
    private Employee employee;

    protected void setUp() { // 테스트가 실행되기 전에 각 테스트 객체에서 실행된다. 이 메소드를 사용하면 테스트에서 사용될 일련의 객체들이 생성된다.
        employee = new Employee("Fred", 0, 10);
        TDate cardDate = new TDate(10, 10, 2000);
        employee.addTimeCard(new TimeCard(cardDate, 40));
    }

    public void testOvertime() {
        TDate newCardDate = new TDate(11, 10, 2000);
        employee.addTimeCard(new TimeCard(newCardDate, 50));
        assertTrue(employee.hasOvertimeFor(newCardDate));
    }

    public void testNormalPay() {
        assertEquals(400, employee.getPay());
    }

    protected void tearDown() { // 끝나면 실행된다.

    }

}
```
> 왜 setup과 teardown이 필요할까? 생성자에서 생성하면 안될까? -> 아직 필요하지 않은 것은 생성하지 않는다면 큰 문제가 되지 않음. 메모리 절약 (및 테스트를 완벽하게 하기위함이 아닐까? 내생각.. 다 클리어하고 setup에서 처리 등..)

### cppUnitLite
cpp 하는사람이나 보면 될듯 생략

### NUnit
C# 하는사람이나 보면 될듯 생략

### 기타 xUnit 프레임워크
다른 플랫폼이나 언어로 이식된 xUnit 버전을 [www.xprogramming.com](www.xprogramming.com)의 섹션에서 다운로드 할 수 있다.

## 일반적인 테스트 하네스
한 번에 여러개의 클레스를 테스트하려면 FIT이나 피트니스를 이용하는 것이 좀 더 적합하다.

### FIT(Framework for Integrated Test)
시스템과 관련된 문서를 작성할 수 있고, 그 문서 내에 입력 값과 출력 값을 기술한 테이블이 포함돼 있으며, 이 문서를 HTML 형식으로 저장할 수 있다면 FIT은 이 문서를 테스트 루틴으로서 실행할 수 있다.  
HTML문서를 읽어 루틴을 실행하고 결과를 출력해준다. 테스트를 통과하면 녹색, 실패하면 빨간색으로 셀의 색이 변경될 것이다.  
FIT 관련 정보를 확인하려면 [http://fit.c2.com](http://fit.c2.com)을 방문

### 피트니스
위키상에 구축된 FIT을 가리킨다.  
[http://fitness.org](http://fitness.org)
[http://docs.fitnesse.org/FrontPage](http://docs.fitnesse.org/FrontPage) 여긴듯??






























