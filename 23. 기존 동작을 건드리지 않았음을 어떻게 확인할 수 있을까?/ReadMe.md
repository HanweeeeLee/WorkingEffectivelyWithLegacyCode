# 23. 기존 동작을 건드리지 않았음을 어떻게 확인할 수 있을까?
코드를 편집할 때 위험을 줄일 수 있는 다양한 기법들을 논의하자.

## 초집중 모두에서 편집하기
키보드 입력 하나하나가 무슨일을 하는지 정확히 아는 것이 바로 프로그래밍의 핵심.  
테스트 주도 개발은 그런 점에서 매우 강력하다. 코드를 테스트 하네스 안에 넣고 테스트 루틴을 1초 내에 실행할 수만 있다면, 영향이 어떻게 미치는지 정확히 알 수 있다.  
테스트 루틴은 초집중 편집을 쉽게 만들어준다. 짝 프로그래밍에도 마찬가지  효과가 있다.  

## 단일 목적 편집
프로그래밍을 할 때, 한 번에 큰 부분을 건드리기 쉽다, 그렇게 되면, 그저 코드가 동작하게 만드는 데 급급해진다.  
페어 프로그래밍을 할 때 "뭐 하고 있어요?" 라고 물어봐달라고 요청하자. 그러면 그 대답하는 한 가지 일을 하도록 하자.

## 서명 유지
실수는 항상 함. 일반적으로 이것을 방지하기 위한 것이 테스트 루틴 사용.  

### ex)
```Java
public void process(List orders, int dailyTargets, double interestRate, int compensationPercent) {
	...
	// 코딩 종료
	...
}
```
이것을 추출함
```Java
public void process(List orders, int dailyTargets, double interestRate, int compensationPercent) {
	processOrders(new orderBatch(orders), new CompensationTarget(dailyTargets, interestRate * 100, compensationPercent))
}
```
이렇게 수정했지만 테스트루틴이 없어서 버그가 발생함.  
  
테스트를 위해 의존 관계를 제거할 떄는 좀 더 주의를 기울여야만 한다. 가능할 때마다 서명 유지를 사용함.  
서명이 바뀌는 것을 피해야 할 때는 전체 메소드 서명을 잘라 붙여서 오류 가능성을 줄일 수 있다.

#### 단계
1. 전체 인수 리스트를 오려내기/복사하기/붙이기 버퍼에 복사한다.
```Java
List orders,
int dailyTargets,
double interestRate,
int compensationPercent
```
2. 그 후 새로운 메소드 선언문을 입력한다.
```Java
private void processOrders() {

}
```
3. 버퍼에 있던 내용을 새로운 메소드 선언문에 붙여 넣는다.
```Java
private void processOrders(List orders,
int dailyTargets,
double interestRate,
int compensationPercent) {

}
```
4. 이어서 새로운 메소드를 위한 호출을 입력한다.
```Java
processOrders();
```
5. 버퍼에 있던 내용을 호출 부분에 붙여 넣는다.
```Java
processOrders(List orders,
int dailyTargets,
double interestRate,
int compensationPercent);
```
6. 최종적으로 매개변수들의 이름만 남기고 이들의 자료형은 모두 지운다.
```Java
processOrders(orders,
dailyTargets,
interestRate,
compensationPercent);
```

## 컴파일러 의존
컴파일러는 자료형 검사 같은것에 이용할 수 있고 작업할 필요가 있는 대상들을 식별하는 데 사용할 수도 있다. 이런게 컴파일러 의존

```cpp
double domestic_exchange_rate;
double foreign_exchange_rate;
```
테스트를 수행하며 이 변수들을 변경할 방법을 찾고 싶다. 따라서 카달로그에서 전역 참조 캡슐화 기법을 사용하도록 결정  
그렇게 하기 위해 선언문들 주위에 클래스를 작성하고, 그 클래스 변수를 선언.
```cpp
class Exchange {
public: 
	double domestic_exchange_rate;
	double foreign_exchange_rate;
};
Exchange exchange;
```
이제 컴파일러를 실행해 컴파일러가 domestic_exchange_rate, foreign_exchange_rate를 찾지 못하는 위치를 검색. 또한 exchange 객체에 접근할 수 있도록 코드를 변경
```cpp
total = domestic_exchange_rate * instrument_shares;
```
-> 
```cpp
total = exchange.domestic_exchange_rate * instrument_shares;
```
  
이 기법을 사용하기 위해서는 컴파일러가 당신을 변경이 필요한 곳으로 보낼 수 있도록 만들어야한다는것.  
컴파일러 의존은 다음 두 가지 단계로 이뤄져있다.
1. 컴파일 오류를 야기하기 위해 선언을 바꾸는 것
2. 그러한 오류들을 찾아 변경하는 것

  
하지만 컴파일러가 버그를 만들 수 있다. 예를들면..
```Java
public int getX() {
	return x;
}
```
모든 사용처를 찾아 주석으로 처리하고 싶다고 가정하자.
```Java
/*
public int getX() {

}
*/
```
하지만 실행했더니 오류가 발생을 하지 않음.  
이것이 getX()가 아무곳에서도 사용되지 않음을 의미할까?  
보장 할 수 없음. 슈퍼 클래스에서 실행되었을 수 있음.

## 짝 프로그래밍
해보자 ㅠㅠ 딴 정보 참조하래....
































