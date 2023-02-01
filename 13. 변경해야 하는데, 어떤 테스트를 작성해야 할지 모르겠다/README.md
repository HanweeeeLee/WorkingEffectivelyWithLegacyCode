# 변경해야 하는데, 어떤 테스트를 작성해야 할지 모르겠다.
일반적으로, 레거시 코드에서 버그를 찾는 것 자체는 문제가 아니다. 하지만 전략적 관점에서 보면, 잘못된 방향일 가능성이 크다.  
그것보다는 개발 팀이 일관되게 올바른 코드를 작성할 수 있도록 도울 수 있는 일을 하는게 더 좋다. 애당초 버그가 발생하지 않도록..

## 문서화 테스트
소프트웨어가 해야 하는 일을 찾은 후 이에 기반한 테스트 루틴을 작성하자. 거의 모든 레거시 시스템에서는 시스템이 '어떻게 동작해야 하는지'보다 '실제로 어떻게 동작하고 있는지'가 더 중요함. 시스템이 해야 하는 일을 바탕으로 테스트 루틴을 작성하는 것은 버그찾기로 돌아가는것. **우리의 목표는 변경 작업을 명확히 하기 위한 테스트 루틴을 정확한 위치에 작성하는것이다.**  
### 기존 동작 유지에 필요한 테스트: 문서화 테스트  
'시스템이 이 작업을 수행해야 한다' 거나 '이 작업을 수행하는 것 같다' 라고 확인하는 테스트를 하는게 아닌 시스템의 현재 동작을 그대로 문서화 하는 테스트  
### 순서
1. 테스트 하네스 내에서 대상 코드를 호출한다.
2. 실패할 것임을 알고 있는 확증문(assertion)을 작성한다.
3. 실패 결과로부터 실제 동작을 확인한다.
4. 코드가 구현할 동작을 기대할 수 있도록 테스트 루틴을 변경한다. 
5. 위 과정을 반복한다.
### 예제
다음 예제에서 PageGenerator 클래스가 "fred"라는 문자열을 생성하지 않을 것임을 알고있다고 하자.
```Java
voic testGenerator() {
	PageGenerator generator = new PageGenerator();
	assertEquals("fred", generator.generate());
}
```
> 당연히 실패. 이로부터 코드의 실제 동작이 어떨지 분명히 나타낼 수 있다. 만약 generator.generate()가 공백 문자열을 만든다고 하면?  
  
```Java
void testGenerator() {
	PageGenerator generator = new PageGenerator();
	assertEquals("", generator.generate());
}
```
> 이제 성공한다. 테스트가 성공했을 뿐 아니라 PageGenerator 클래스에 대한 가장 기본적인 사실 중 하나를 문서화한 것이기도 하다. 그것은 PageGenerator 객체를 생성하고 곧바로 generate 메소드를 호출하면 공백 문자열이 반환된다는 점이다.  
  
```Java
void testGenerator() {
	PageGenerator generator = new PageGenerator();
	generator.assoc(RowMappings.getRow(Page.BASE_ROW)); // 새로 추가
	assertEquals("fred", generator.generate());
}
```
> 돌려보았더니 실패함. 테스트 하네스의 오류 메시지로부터 결과 문자열이 다음과 같으므로 테스트 루틴에서 기대하는 결과 값으로 해당 문자열을 설정한다.
  
```Java
void testGenerator() {
	PageGenerator generator = new PageGenerator();
	generator.assoc(RowMappings.getRow(Page.BASE_ROW));
	assertEquals("<node><carry>1.1 vectrai</carry></node>", generator.generate());
}
```
> 이제 성공
  
이게 테스트? 라고 생각할 수 있겠지만 이러한 테스트는 소프트웨어가 반드시 따라야 할 규칙에 따라 작성된 것이 아니다. 지금은 버그를 찾는 것이 목적이 아니고, 시스템의 현재 동작과의 차이라는 형태로 나타나는 버그를 나중에 발견하기 위한 구조를 만드는것이 목적  
여기서의 테스트는 당위론을 따르는 것이 아니라, 시스템의 실제 동작을 그대로 문서화 한것이 된다.  
시스템 한 부분의 동작을 이해할 수 있다면, 그 지식과 시스템에 새롭게 기대되는 동작에 관한 지식을 함께 사용해 변경 작업을 수행할 수 있게 된다.  
이렇게 하는게 손으로 일일이 디버그 해서 파악하는것보다 안전하고 빠름..  

## 클래스 문서화 
어떤 클래스를 대상으로 무엇을 테스트해야 할 지 결정하고 싶다. 어떻게 해야할까?
1. 로직이 엉켜 있는 부분을 찾는다. 코드에 이해할 수 없는 부분이 있다면, 감지 변수를 사용해 해당 부분을 문서화할 것을 검토한다. 코드의 특정 부분이 실행되는지 확인하기 위해 감지 변수를 사용한다.
2. 클래스나 메소드의 책임을 파악했으면, 일단 적업을 멈춘 후 실패를 일으킬 수 있는 것들의 목록을 만든다. 그리고 그러한 실패를 일으키는 테스트 루틴을 작성할 수 있는지 고려한다.
3. 테스트 루틴에 전달되는 입력 값을 검토한다. 극단적인 값이 주어질 경우 어떤 일이 벌어질지 확인해야 한다.
4. 객체가 살아이있는 동안 항상 참이어야 하는 조건이 있는지 확인한다. 이런 조건을 불변 조건이라고 부른다. 불변 조건을 검증하기 위한 테스트 루틴을 작성해보자. 불변 조건을 발견하기 위해 클래스를 리팩토링해야 할 수도 있다. 리팩토링을 통해 코드의 바람직한 모습에 대한 새로운 지식을 얻게 되기도 한다. 
    
문서를 읽을 사람에게 중요한 것이 무엇인지에 집중해보자.

## 정해진 목표가 있는 테스트
코드의 특정 부분을 이해하기 위한 테스트 루틴을 작성했으면, 변경 대상의 범위를 살펴보고 테스트 루틴이 그 범위를 포함하고 있는지 조사해야 한다. 다음의 자바 클래스 메소드는 탱크 내의 연료량을 계산한다.
```Java
public class FuelShare {
	private long cost = 0;
	private double corpBase = 12.0;
	private ZonedHawthorneLease lease;
	...

	public voic addReading(int gallons, Date readingDate) {
		if (lease.isMonthly()) {
			if (gallons < Lease.CORP_MIN) {
				cost += corpBase;
			} else {
				cost += 1.2 * priceForGallons(gallons);
			}
		}
		...
		lease.postReading(readingDate, gallons);
	}
	...
}
```
> FuelShare 클래스를 직접 변경하고 싶다고 하자. 변경사항은 다음과 같다. 최상위 if문 전체를 새로운 메소드로 추출한 후 ZonedHawthorneLease 클래스로 옮긴다. lease 변수는 ZonedHawthorneLease 클래스의 인스턴스이다. 
  
리팩토링 한 다음 코드
```Java
public class FuelShare {
	public void addReading(int gallons, Date readingDate) {
		cost += lease.computeValue(gallons, priceForGallons(gallons));
		...
		lease.postReading(readingDate, gallons);
	}
	...
}

public class ZonedHawthorneLease extends Lease {
	public long computeValue(int gallons, long totalPrice) {
		long cost = 0;
		if (lease.isMonthly()) {
			if (gallons < Lease.CORP_MIN) {
				cost += corpBase;
			} else {
				cost += 1.2 * totalPrice;
			}
		}
		return cost;
	}
	...
}
```
> 리팩토링이 잘 이루어졌는지 확인하는 테스트 코드를 작성하려면 어떻게? 확실한 점은 다음의 로직은 전혀 변경되지 않을것이다.
```Java
if (gallons < Lease.CORP_MIN) {
	cost += corpBase;
}
```
다음의 else문은 다음과 같이 변경이 된다.
```Java
else {
	// cost += 1.2 * priceForGallons(gallons);
	cost += 1.2 * totalPrice;
}
```
이 else문은 어딘가의 테스트 루틴에서 실행되야 한다.

```Java
public void testValueForGallonsMoreThanCorpMin() {
	StandardLease lease = new Standardlease(Lease.MONTHLY);
	FuelShare share = new FuelShare(lease);

	share.addReading(FuelShare.CORP_MIN + 1, new Date());
	assertEquals(12, share.getCost());
}
```
> 이처럼 조건 분기를 문서화할 때는 자신이 입력한 값이 코드의 특이한 동작을 일으켜서 원래 실패해야 할 테스트가 성공하는 일이 없는지 파악하는 것이 중요하다. 
  
금액 표시에 int가 아닌 double형이 사용된다고 가정하자.
```Java
public class FuelShare {
	private double cost = 0;
	...

	public voic addReading(int gallons, Date readingDate) {
		if (lease.isMonthly()) {
			if (gallons < Lease.CORP_MIN) {
				cost += corpBase;
			} else {
				cost += 1.2 * priceForGallons(gallons);
			}
		}
		...
		lease.postReading(readingDate, gallons);
	}
	...
}
```
입력값을 신중히 선택하지 않을 경우 메소드를 추출할 때 잘못해도 이를 인지하지 못함. 값 절삭..  
예를들면 Lase.CORP_MIN 상수의 값이 10 이라면
```Java
public void testValue() {
	StandardLease lease = new Standardlease(Lease.MONTHLY);
	FuelShare share = new FuelShare(lease);

	share.addReading(1, new Date());
	assertEquals(12, share.getCost());
}
```
> 1은 10보다 작으므로 초깃값 0에 12.0을 더하게 된다 따라서 cost의 결과 값은 12.0이 된다. 하지만 다음과 같이 메소드를 추출할 때 cost 변수를 double이 아닌 int형으로 선언하면 어떻게 될까?
```Java
public class ZonedHawthorneLease extends Lease {
	public long computeValue(int gallons, long totalPrice) {
		long cost = 0;
		if (lease.isMonthly()) {
			if (gallons < Lease.CORP_MIN) {
				cost += corpBase;
			} else {
				cost += 1.2 * totalPrice;
			}
		}
		return cost;
	}
	...
}
```
> 항상 문제가 발생하지는 않지만 가끔은 문제가 발생할 경우가 있다.
  
이 문제를 어떻게 해결할까?
1. 손으로 직접 계산해본다. 
2. 디버거를 사용해 매 단게 별로 입력 값의 형 변환을 추적한다.
3. 감지 변수를 사용해 특정 경로가 실행되고 형 변환이 일어나는지 검증한다.
4. 좀 더 작은 크기의 코드 단위로 문서화한다.

## 문서화 테스트를 작성하기 위한 경험칙
1. 변경 대상 부분을 위한 테스트 루틴을 작성한다. 코드의 동작을 이해하는 데 핑요해 보이는 테스트 케이스를 가급적 많이 작성한다.
2. 테스트 루틴을 작성한 후, 변경하려는 구체적인 코드들을 조사하고, 이에 관련된 테스트 루틴을 작성한다.
3. 기능을 추출하거나 이동할 경우, 기존의 동작이나 동작 간의 관계를 검증하는 테스트를 개별적으로 작성한다. 이를 통해 이동 대상 코드가 실행되는지, 그리고 적절히 관계를 유지하는지 검증한다. 검증이 끝난 후에 코드를 이동한다.























