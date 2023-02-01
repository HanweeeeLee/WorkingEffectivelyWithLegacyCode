# 12. 클래스 의존관계, 반드시 없애야 할까?

## 교차 지점
교차 지점은 특정 변경에 의한 영향을 감지할 수 있는 프로그램 내의 위치를 말한다.  
우선 변경이 필요한 위치들을 확인한 후, 이 지점들이 외부에 미치는 영향을 추적하여 교차지점을 찾자.  
영향이 탐지되는 모든 위치가 교차지점이지만, 모든 교차지점이 최상의 교차지점은 아니다. 최상의 교차지점을 찾아보자.

### 간단한 경우
금액 계산 방법을 변경하기 위해 Invoice라는 자바 클래스를 변경하고 싶다고 하자. 모든 비용을 계산하는 것은 getValue 메소드다.
```Java
public class Invoice {
	...
	public Money getValue() {
		Money total = itemsSum();
		if (billingDate.after(Date.yearEnd(openingDate))) {
			if (originator.getState().equals("FL") || originator.getState().equals("NY")) {
				total.add(getLocalShipping()); 
			} else {
				total.add(getDefaultShipping());
			}
		} else {
			total.add(getSpanningShipping());
		}
		total.add(getTax());
		return total;
		...
	}
}
```
뉴욕으로 보내지는 운송 비용의 계산 방법을 변경해야 하는 상황이다. 세법 변경으로 세금이 추가됐고 이로 인해 증가된 화물 출하 비용을 고객에게 청구해야만한다.  
이를 반영하기위해 출하 비용의 계산 로직을 추출해서 신규 클래스인 ShippingPricer를 작성했다.
```Java
public class Invoice {
	pulbic Money getValue() {
		Money total = itemsSum();
		// if (billingDate.after(Date.yearEnd(openingDate))) {
		// 	if (originator.getState().equals("FL") || originator.getState().equals("NY")) {
		// 		total.add(getLocalShipping()); 
		// 	} else {
		// 		total.add(getDefaultShipping());
		// 	}
		// } else {
		// 	total.add(getSpanningShipping());
		// }
		total.add(shippingPricer.getPrice());
		total.add(getTax());
		return total;
	}
}
```
원래 getValue() 메소드가 수행했던 작업 대부분이 shippingPricer로 넘어갔다.  
교차 지점을 찾으려면 변경 위치에서부터 영향 추적을 시작해야한다.
![KakaoTalk_Photo_2023-01-26-17-47-23](https://user-images.githubusercontent.com/60125719/214793455-7a299b2c-8234-46e8-8b2f-4846a47d2d14.jpeg)
> getValue메소드는 invoice 클래스 내에서는 전혀 사용되지 않고, BillingStatement클래스 내의 makeStatement 메소드에서 사용된다.  
  
생성자도 수정해야 하므로 생성자에 의존하는 코드도 살펴보자. 생성자 내에서 ShippingPricer 객체를 생성하는데, 이 객체는 자신을 사용하는 메소드를 제외하고 아무것에도 영향을 미치지 않는다. 그리고 이 객체를 사용하는 메소드는 getValue가 유일하다.
![KakaoTalk_Photo_2023-01-26-17-51-47 001](https://user-images.githubusercontent.com/60125719/214794339-4c3d6242-ea16-4e33-9491-f0cde3925390.jpeg)

종합해보면
![KakaoTalk_Photo_2023-01-26-17-51-48 002](https://user-images.githubusercontent.com/60125719/214794406-ee06de4f-4afa-49b3-b0dc-09ad24ed5e87.jpeg)

그럼 교차점은?  
이 그림에서 타원으로 표시된 것들은 우리가 접근할 수만 있다면 모두 교차점임  
여기서 가장 최선의 교차지점은 Invoice의 getValue를 실행하고 그 반환 값을 검사하는 테스트 루틴을 작성하는것이다. 이렇게 하면 작업량을 줄일 수 있음

### 상위 수준의 교차 지점
대부분의 경우 변경 작업을 위한 가장 좋은 교차 지점은 변경 대상 클래스의 public 메소드중 하나이다. 그런데 이곳이 아닐 때가 있음  
Invoice 클래스에서 운송 비용의 계산 방법을 변경하고, 운송 업체를 관리하기 위한 필드를 포함하도록 Item이라는 이름의 클래스를 변경해야한다고 치자. 또 BillingStatement 클래스의 처리를 운송 업체별로 구분할 필요도 있다.  
![KakaoTalk_Photo_2023-01-26-18-07-06](https://user-images.githubusercontent.com/60125719/214797445-8b60239a-c280-4b07-84e9-f0ccee2c78a1.jpeg)
이 클래스들에 대한 테스트 루틴이 없다면, 각 클래스마다 개별적으로 테스트 루틴을 작성하고 필요한 변경을 수행하는 방법을 생각할 수 있다. 그런데 이것보다 상위 수준의 교차 지점이 발견된다면 변경 작업을 좀 더 효율적으로 진행할 수 있다.  
장점:  
1. 의존 관계를 그리 많이 제거하지 않아도 된다
2. 코드를 묶음 단위로 취급할 수 있다.  
  
```Java
voic testSimpleStatement() {
	Invoice invoice = new Invoice();
	invoice.addItem(new Item(0, newMoney(10)));
	BillingStatement statement = new BillingStatement();
	statement.addInvoice(invoice);
	assertEquals("", statement.makeStatement());
}
```
이 테스트 루틴에 의해 BillingStatement가 한 개의 품목(item)을 갖는 송장(Invoice)에 대해 어떤 텍스트를 생성하는지 확인한 후, 실제로 이 텍스트를 사용하도록 테스트 루틴을 변경할 수 있다.  
여기서 BillingStatement가 이상적인 교차 지점인 이유는 무엇일까? 클래스들의 변경에 의한 영향을 검출하는 데 사용될 수 있는 유일한 곳이기 때문
![KakaoTalk_Photo_2023-01-26-18-15-00](https://user-images.githubusercontent.com/60125719/214798969-1b92fb36-5d13-4550-a68d-e6f1f5e967b4.jpeg)
> 모든 영향을 makeStatement를 통해 검출할 수 있다는 점에 주목하자. 이런 지점을 조임지점이라고 한다고 함  
  
조임지점은 변경 지점에 의해 결정된다는 것을 명심하자. 여러 곳에서 호출되는 클래스일지라도, 이 클래스의 다수 변경에 대한 조임 지점은 한 개뿐일 수 있다. 송장시스템을 한번 보면..

![KakaoTalk_Photo_2023-01-26-18-18-04](https://user-images.githubusercontent.com/60125719/214799550-3a84f86c-6c3e-44c4-9742-968391064d00.jpeg)
지금까지 논의된 적은 없지만 Item 클래스는 needsReoder라는 메소드도 가지고 있다. 이 메소드는 InventoryControl 객체가 발주 여부를 판단할 때 호출된다. 이 변경 떄문에 영향 스케치를 수정? ㄴㄴ  
item 에 shippingCarrier필드를 추가하더라도 needsReorder 메소드에 아무 영향도 미치지 못한다. 그러므로 BillingStatement는 여전히 조임 지점으로 테스트에 적함한 지점이다.  
  
좀 더 시나리오를 변경하여 Item클래스에 공급자(supplier)를 얻어오고 설정하는 메소드를 추가해야 한다고 하자. InventoryControl과 BillingStatement 클래스는 공급자의 이름을 사용한다.
![KakaoTalk_Photo_2023-01-26-18-23-43](https://user-images.githubusercontent.com/60125719/214800692-6cf81a11-b54b-4ea4-972e-874b7cafb65e.jpeg)
> 교차지점은 더 이상 한개가 아니다. 하지만 run메소드와 makeStatement 메소드를 묶어서 보면 한 개의 교차 지점으로 볼 수 있다.  
  
조임 지점을 금방 찾을 수 있지만 아닐 수 있음 이럴땐 어떻게 해야할까?  
1. 변경 지점으로 다시 돌아가보자. 어쩌면 한 번에 너무 많은 변경을 시도하는 것일수도 있다.. 가급적 개별 변경 지점의 근처에서 테스트 루틴을 작성하려고 노력하자.
2. 영향 스케치 내에서 공통적인 사용 방법을 찾는다. 예를 들어 한 개의 메소드나 변수가 세 곳에서 사용된다고 해서, 이것이 반드시 세 가지 방법으로 사용되고 있음을 의미하지 않는다. "이 메소드를 분리하면 이곳에서 변경을 감지할 수 있는가?"라고 질문을 해보자. 메소드가 비교 가능한 값을 갖는 객체와 동일한 방법으로 사용된다면 한곳에서만 테스트해도 된다. 

## 조임 지점을 이용한 설계 판단
조임 지점이 어디에 있는지 주목하면 코드를 개선하는 방법에 대한 힌트를 얻을 수 있다.  
조임 지점이란 실제로 무엇일까? 조임 지점은 자연적인 캡술화의 경계다. (BillingStatement 클래스의 makeStatement 메소드가 송장과 품목에 대한 조임 지점이라면, makeStatement 메소드의 결과가 예상과 다를 경우 어느 곳을 확인해야 할지 알 수 있다.)

## 조임 지정의 함정
단위 테스트를 작성할 떄 곤란한 상황에 처하는 경우가 있다. 그중 하나는 단위 테스트가 점점 소규모 통합 테스트로 커지는 것이다.  
새로운 코드를 위한 테스트를 작성할 떄의 핵심은 가급적 독립적으로 클래스를 테스트 하는 것.  
단위 테스트의 목적은 객체들이 전체적으로 올바르게 동작하는지 확인하는 것이 아니라 하나의 객체가 어떻게 동작하는지 확인하는 것이라는걸 기억해야함.  
따라서 필요하다면 협업 클래스를 위장해서 가짜 클래스로 사용하기도 하고..  
  
기존 코드를 위한 테스트 루틴을 작성할 경우에는 좀 다름. 애플리케이션의 일부를 추출해서 그 부분에 대한 테스트 루틴을 작성하는것이 이득일 수 있다.  
  
조임 지점의 테스트 루틴은 숲에 걸어 들어가서 줄을 그은 다음 '이 구역은 전부 내 것'이라고 말하는것과 비슷함.



























