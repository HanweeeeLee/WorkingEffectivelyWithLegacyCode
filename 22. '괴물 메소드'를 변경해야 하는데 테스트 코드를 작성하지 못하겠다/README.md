# '괴물 메소드'를 변경해야 하는데 테스트 코드를 작성하지 못하겠다
대규모 메소드는 다루기 힘든 수준이라면, 괴물 메소드는 재앙이라고 부를 만하다.  
괴물 메소드란 너무나 길고 복잡해서 손대고 싶지 않은 메소드를 의미하는데 이정도의 메소드를 다룰때는 어디부터 손대야 할까?

## 괴물 메소드의 다양한 종류
### 불릿 메소드
불릿 메소드란 들여쓰기가 거의 돼 있지 않은 메소드를 말한다.  
코드 덩어리 내의 일부 코드는 들여쓰기가 돼 있을지 모르지만, 메소드 자체는 들여쓰기가 돼 있지 않다.
```Java
void Reservation::extend(int additionalDays)
{
  int status = RIXInterface::checkAvailable(type, location, startingDate);
  
  int identCookie = -1;
  switch(status) {
    case NOT_AVAILABLE_UPGRADE_LUXURY:
      identCookie = RIXInterface::holdReservation(Luxury,location,startingDate,additionalDays +dditionalDats);
      break;
    case NOT_AVAILABLE_UPGRADE_SUV:
    {
      int theDays = additionalDays + additionalDays;
      if(RIXInterface::getOpCode(customerID) != 0)
        theDays++;
      identCookie = RIXInterface::holdReservation(SUV,location,startingDate,theDats)
    }
    break;
    case NOT_AVAILABLE_UPGRADE_VAN:
      identCookie = RIXInterface::holdReservation(Van,location,startingDate, addtionalDats + additionalDats);
      break;
    case AVAILABLE:
    default:
      RIXInterface::holdReservation(type,location,startingDate);
      break;
  }
  
  if(identCookie != -1 && state == Initial) {
    RIXInterface::waitlistReservation(type,location,startingDate);
  }
  
  Customer c = res_db.getCustomer(customerID);
  
  if(c.vipProgramStatus == VIP_DIAMOND) {
    upgradeQuery = true;
  }
  
  if(!upgradeQuery)
    RIXInterface::extend(lastCookie, days + additionalDays);
  else {
    RIXInterface::waitlistReservation(type,location,startingDate);
    RIXInterface::extend(lastCookie, days + additionalDats +1);
  }
  ...
}
```
위 코드는 불릿 메소드의 일반적인 형태인데, 운이 좋으면 누군가 구별하기 쉽도록 단락 사이에 공백 행을 추가했거나 주석을 달아놓았을지도 모른다.  
긱 딘릭별로 메소드를 추출할 수 있다면 이상적이지만, 그렇게 쉽게 리팩토링 가능한 경우는 거의 없다.  
불릿 메소드는 그나마 다른 종류의 괴물 메소드 보다는 낫다. 엉망인 들여쓰기를 제외하면 코드 흐름을 추적할 만하기 때문이다.  

### 혼잡 메소드
혼잡 메소드란 들여쓰기가 된 한개의 대규모 단락으로 구성된 메소드를 말한다. 가장 단순한 예시는 한개의 대규모 조건문을 갖는 메소드다.  
대부분의 메소드는 완전한 불릿 메소드나 혼잡 메소드가 아니라 그 중간쯤의 형태를 보여준다.  
상당 수의 혼잡 메소드는 깊게 중첩된 곳에 대규모의 불릿 단락이 숨어 있는데, 내부에 중첩되어 있어 동작 확인을 위한 테스트 루틴 작성이 어렵다.

## 리팩토링 자동화 도구를 사용해 괴물 메소드 공략하기
메소드 추출 도구를 갖고 있다면, 도구로 할 수 있는 것과 할 수 없는 것을 분명히 인지해야 한다.  
최근의 리팩토링 도구들은 대부분 간단한 메소드 추출 등의 다양한 리팩토링 작업을 지원하지만, 대규모 메소드 분할 시에 요구되는 보조적 리팩토링을 모두 지원하지는 않는다.  
현재의 리팩토링 도구들은 재배치된 결과가 안전한지 분석 할 수 없고, 이는 곧 버그의 원인이 될 수도 있다.  
대규모 매소드에 리팩토링 도구를 효과적으로 사용하려면, 그 도구만으로 일련의 변화들을 수행하고 그 외의 소스는 전혀 손대지 말아야 한다.  
추출을 수행하는 주요 목적은 다음과 같이 두 가지다.
1. 이상한 의존 관계로부터 로직을 분리한다.
2. 이후의 리팩토링을 위해 테스트 루틴을 작성하기 쉽도록 봉합부를 작성한다.

```Java
class CommoditySelectionPanel
{
  ...
  public void update() {
    if (commodities.size() > 0 && commodities.GetSource().equals("local")) {
      listbox.clear();
      for(Iterator it = commodities.iterator();it.hasNext();) {
        Commodity current = (Commodity)it.next();
        if(commodity.isTwilight() && !commodity.match(broker))
          listbox.add(commodity.getView());
      }
    }
    ...
  }
  ...
}
```
이 메소드에서는 개선할 만한 점이 다수 있는데, 무엇보다 이상하게도 이런 종류의 필터링 작업을 Panel 클래스에서 수행하고 있는데 이 클래스는 화면 표시를 책임지는 것이 이상적이기 때문이다.  
이 코드를 풀어내는 것은 어렵다. 현재 상태에서 이 메소드에 대한 테스트 루틴을 작성하려고 하면 리스트 박스의 상태에 대해 작성할 수 있겠지만, 이로 인해 설계가 개선될 것이라고는 도저히 생각할 수 없다.  
리팩토링 도구를 사용할 수 있다면, 메소드 내 코드 덩어리의 상당 부분에 이름을 붙이는 것과 함께 의존관계를 제거하는 작업도 시작할 수 있다.  
몇차례 추출하고 난 후의 코드는 다음과 같다.  
```Java
class CommoditySelectionPanel
{
  ...
  public void update() {
    if(commoditiesAreReadyForUpdate()) {
      clearDisplay();
      updateCommodities();
    }
    ...
  }
  
  private boolean commoditiesAreReadyForUpdate() {
    return commodities.size() > 0 && commodities.GetSource().equals("local");
  }
  
  private void clearDisplay() {
    listbox.clear();
  }
  
  private void updateCommodities() {
    for(Iterator it = commodities.iterator(); it.hasNext();) {
        Commodity current = (Commodity)it.next();
        if(singleBrokerCommodity(commodity)) {
          displayCommodity(current.getView());
        }
      }
  }
  
  private boolean singleBrokerCommodity(Commodity commodity) {
    return commodity.isTwilight() && !commodity.match(broker);
  }
  
  private void displayCommodity(CommodityView view) {
    listbox.add(view);
  }
  ...
  
}
```
update메소드 내의 코드는 그다지 달라지지 않았지만 처리작업이 다른 메소드로 위임 됐다.  
update 메소드는 기존 코드의 뼈대처럼 되었고 이름은 다소 부자연스러워 보여도 출발점으로 나쁘지는 않다.  
최소한 코드 내용이 상위 수준에서 전달되며 의존 관계를 제거하기 위한 봉합점도 제공한다.  
메소드 이름은 어느정도 정리가 끝난후에 바꿀수 있고, 일련의 리팩토링을 마치고 나면 설계는 다음과 같이 된다.  
![KakaoTalk_Photo_2023-04-02-15-55-06](https://user-images.githubusercontent.com/50142323/229337480-67a059f7-8861-4c22-90f0-30bf54027122.jpeg)

자동화 도구를 사용한 메소드 추출 시에 잊으면 안 되는 것은 다수의 조잡한 작업들을 안전하게 수행할 수 있다는 점과 세부 처리를 테스트 코드 작성 후로 미룰 수 있다는 점이다.  

## 수동 리팩토링에 도전
리팩토링 도구가 없을 때는 직접 정확성을 검증해야 하는데, 이떄 리팩토링 도구를 대신할 수 있는 것이 테스트 루틴이다.  
괴물 메소드는 테스트 루틴의 작성이나 리팩토링, 기능추가 작업이 매우 어렵다. 하지만 이때 사용할 수 있는 몇가지 기법이 있다.  
이 기법을 설명하기 전에 메소드 추출 과정에서 실수하기 쉬운 사항들을 살펴보자.  

1. 추출한 메소드에 변수를 전달하는 것을 잊기 쉽다

### 감지 변수 도입
### 아는 부분 추출하기
### 의존 관계 이삭줍기
### 메소드 객체 추출

## 전략
### 뼈대 메소드
### 처리 시퀀스 발견
### 우선 현재 클래스 내에서 추출
### 작은 조각 추출
### 추출을 다시 할 각오
