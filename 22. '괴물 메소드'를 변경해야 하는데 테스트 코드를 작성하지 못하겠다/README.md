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
긱 단락별로 메소드를 추출할 수 있다면 이상적이지만, 그렇게 쉽게 리팩토링 가능한 경우는 거의 없다.  
불릿 메소드는 그나마 다른 종류의 괴물 메소드 보다는 낫다. 엉망인 들여쓰기를 제외하면 코드 흐름을 추적할 만하기 때문이다.  

### 혼잡 메소드
혼잡 메소드란 들여쓰기가 된 한개의 대규모 단락으로 구성된 메소드를 말한다. 가장 단순한 예시는 한개의 대규모 조건문을 갖는 메소드다.  
```Java
Reservation::Reservation(VehicleType type, int customerID, long startingDate, int days, XLocation l): type(type), custimerID(customerID), startingDate(startingDate), days(days), lastCookie(-1), state(Initial), tempTotal(0)
{
  location = l;
  upgradeQuery = false;
  
  if(!RIXInterface::available()) {
    RIXInterface::doEvents(100);
    PostLogMessage(0, 0, "delay on reservation creation");
    int holdCookie = -1;
    switch(status) {
      case NOT_AVAILABLE_UPGRADE_LUXURY:
        holdCookie = RIXInterface::holdReservation(Luxury,l,startingDate);
        if(holdCookie != 9;) {
          holdCookie |= 9;
        }
        break;
      case NOT_AVAILABLE_UPGRADE_SUV:
        holdCookie = RIXInterface::holdReservation(SUV,l,startingDate);
        break;
      case NOT_AVAILABLE_UPGRADE_VAN:
        holdCookie = RIXInterface::holdReservation(Van,l,startingDate);
        break;
      case AVAILAVLE:
      default:
        RIXInterface::holdReservation;
        state = Held;
        break;
    }
  }
  ...
}
```
진정한 혼잡 메소드인지 알 수 있는 가장 좋은 방법은 대규모 메소드 내의 블록들을 정렬해보는 것이다. 어질어질하다면 그건 혼잡 메소드이다.
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

1. 추출한 메소드에 변수를 전달하는 것을 잊기 쉽다.
2. 추출된 메소드에 붙인 이름이 기초 클래스 내에 있는 동일한 이름의 메소드를 은폐하거나 재정의할 가능성이 있다.
3. 매개변수를 전달하거나 반환값을 대입할 때 실수를 저지르기 쉽다.

잘못을 저지를 가능성은 이외에도 다분한데 다음 기법을 사용하면, 위험성을 어느 정도 감소시킬 수 있다.

### 감지 변수 도입
리팩토링할때 배포코드에는 기능을 추가하면 안되지만, 그렇다고 해서 코드를 절대 추가하면 안된다는 것은 아니다.  
클래스에 추가된 변수를 이용해 리팩토링 대상 메소드의 상태를 감지할 수 있는 경우도 있다.  
리팩토링을 완료한 후 이 변수를 제거하면 코드는 깔끔한 상태로 되돌아가는데, 이를 가리켜 감지 변수 도입이라고 부른다.  
다음을 살펴보자.  
```Java
public class DOMBuilder
{
  ...
  void processNode(XDOMNSnippet root, List childNodes)
  {
    if(root != null) {
      if(childNodes != null)
        root.addNode(new XDOMNSnippet(childNodes));
      root.addChild(XDOMNSnippet.NullSnippet);
    }
    List paraList = new ArrayList();
    XDOMNSnippet snippet = new XDOMNReSnippet();
    snippet.setSource(m_state);
    for(Iterator it = childNodes.iterator(); it.hasNext();) {
      XDOMNNode node = (XDOMNNode)it.next();
      if(node.type() == TF_G || node.type() == TF_H || (node.type() == TF_GLOT && node.isChild())) {
        paraList.addNode(node);
      }
      ...
    }
    ...
  }
  ...
}
```
processNode 메소드 내의 많은 처리들이 XDOMNSnippet 객체에 대해 실행되고 있는것으로 보이는데, 따라서 이 메소드에 적절한 인수만 전달된다면 어떤 테스트 루틴도 작성할 수 있을것이다.  
그러나 실제로는 많은 복잡한 처리들이 내부적으로 수행되기 때문에 그 내용을 상당히 간접적으로만 알 수 있다.  
이런 상황일 때 작업을 편하게 하기 위해 감지 변수를 도입할 수 있다.  
```Java
public class DOMBuilder
{
  public boolean nodeAdded = false;   //변수 추가
  ...
  void processNode(XDOMNSnippet root, List childNodes)
  {
    if(root != null) {
      if(childNodes != null)
        root.addNode(new XDOMNSnippet(childNodes));
      root.addChild(XDOMNSnippet.NullSnippet);
    }
    List paraList = new ArrayList();
    XDOMNSnippet snippet = new XDOMNSnippet();
    snippet.setSource(m_state);
    for(Iterator it = childNodes.iterator(); it.hasNext();) {
      XDOMNNode node = (XDOMNNode)it.next();
      if(node.type() == TF_G || node.type() == TF_H || (node.type() == TF_GLOT && node.isChild())) {
        paraList.add(node);
        nodeAdded = true;
      }
      ...
    }
    ...
  }
  ...
}
```
예를 들어 다음과 같이 특정 타입의 노드만이 paraList에 추가 됐음을 확인하기 위해 인스턴스 변수를 도입할 수 있다.  
이 변수가 존재하더라도 해당 조건을 만족하는 입력값을 작성해야 한다. 입력값을 작성 할 수 있다면, 해당 로직 부분을 추출해서 변경 전과 동일하게 테스트에 통과함을 확인 할 수 있다.  
```Java
//노드타입이 TF_G일 경우에 노드가 추가된 것을 확인하는 테스트 루틴
void testAddNodeOnBasicChild()
{
  DOMBuilder builder = new DomBuilder();
  List children = new ArrayList();
  children.add(new XDOMNNode(XDOMNNode.TF_G));
  Builder.processNode(new XDOMNSnippet(), children);
  
  assertTrue(Builder.nodeAdded);
}

//잘못된 노드 타입일 경우 노드가 추가되지 않았음을 확인하는 테스트 루틴
void testNoAddNodeOnNonBasicChild()
{
  DOMBuilder builder = new DomBuilder();
  List children = new ArrayList();
  children.add(new XDOMNNode(XDOMNNode.TF_A));
  Builder.processNode(new XDOMNSnippet(), children);
  
  assertTrue(!Builder.nodeAdded);
}
```
이러한 테스트 루틴들이 있으면, 노드 추가여부를 판단하는 조건문을 안심하고 추출 할 수 있다.  
조건문 전체를 복사한 후, 앞서 작성했던 테스트 루틴을 통해 조건과 일치할 때 노드가 추가되는 것을 확인할 수 있다.
```Java
public class DOMBuilder
{
  void processNode(XDOMNSnippet root, List childNodes)
  {
    if(root != null) {
      if(childNodes != null)
        root.addNode(new XDOMNSnippet(childNodes));
      root.addChild(XDOMNSnippet.NullSnippet);
    }
    List paraList = new ArrayList();
    XDOMNSnippet snippet = new XDOMNSnippet();
    snippet.setSource(m_state);
    for(Iterator it = childNodes.iterator(); it.hasNext();) {
      XDOMNNode node = (XDOMNNode)it.next();
      if(isBasicChild(node)) {
        paraList.add(node);
        nodeAdded = true;
      }
      ...
    }
    ...
  }
  private boolean isBasicChild(XDOMNNode node) {
    return node.type() == TF_G || node.type() == TF_H || (node.type() == TF_GLOT && node.isChild());
  }
  ...
}
```
감지 변수를 사용할 때는 일련의 리팩토링을 수행하는 동안 클래스에 남겨두고, 모든 리팩토링을 마친 후 삭제하는 편이 좋다.  

### 아는 부분 추출하기
괴물 메소드를 다루는데 도움이 되는 또 다른 전략으로서 작은것부터 시작하는 것이 있다.  
테스트 루틴 없이도 자신 있게 추출할 수 있는 작은 코드 조각을 찾아서 이 코드에 대한 테스트 코드를 작성한다.  
필자의 기준의 작은 코드란 두세줄에서 기껏해야 다섯 줄 정도며 쉽게 붙일 수 있는 코드 덩어리를 말한다.  
이런 소규모 추출을 할때 주의할 점은 추출의 결합 계수인데, 결합 계수란 추출 대상 메소드에 드나드는 값의 총 개수를 말한다.  
```Java
void process(int a, int b, int c) {
  int maximum;
  if(a > b)
    maximum = a;
  else
    maximum = b;
  ...
}
```
예를 들어 위 메소드에서 max메소드를 추출할때의 결합 계수는 3이 된다. 추출 후의 코드는 다음처럼 바뀐다.
```Java
void process(int a, int b, int c) {
  int maximum = max(a,b);
  ...
}
```
max 메소드는 두개의 변수가 입력되고 한개의 변수는 출력되므로 결합 계수는 3이다.  
일반적으로 결합 계수가 작은 추출이 바람직한데, 실수할 염려가 적기 때문이다.  
어떤 매개변수도 받지 않고 어떤 값도 반환하지 않는 메소드를 추출하는 것만으로 괴물 메소드에 대한 대처는 많은 진전을 이룰 수 있다.  
코드 덩어리에 이름을 짓다보면, 그 덩어리가 어떤 것이고 객체에 어떤 영향을 미치는지 점점 이해하게 되고, 한곳에서 이해하고 나면 이것이 점점 확장돼서   
생산성이 더 높은 별도의 관점에서 설계를 바라볼 수 있게 된다.  

### 의존 관계 이삭줍기
메소드의 주요 목적과는 거리가 먼 부가적인 코드가 괴물 메소드 내에 들어 있을 수 있는데, 이런 코드는 필요하긴 하지만 그리 복잡하지 않기 때문에 실수로 손상을 입혀도 금세 알 수 있다.  
하지만 그렇다고 해도 자칫 메소드의 주요 로직을 무너뜨릴지도 모르는 위험을 쉽게 감수할 수는 없는데 이런 경우에 의존 관계 수집 기법을 사용할 수 있다.  
먼저 변경되면 안되는 로직을 확인하기 위한 테스트 루틴을 작성한 후 테스트 대상이 아닌 부분을 추출한다.  
이러면 적어도 중요 동작은 유지될 것이라 확신할 수 있다.
```Java
void addEntry(Entry entry) {
  if(view != null && DISPLAY == true) {
    view.show(entry);
  }
  ...
  if(entry.category().equals("single") || entry.category("dual")) {
    entries.add(entry);
    view.showUpdate(entry, view.GREEN);
  } else {
    ...
  }
  
}
```
이 경우에는 메소드에 대한 테스트 루틴을 작성해 올바른 조건에서 항목이 추가되는지 검증할 수 있다.  
그 후에 모든 동작을 아우른다는 확신이 들면, 화면 표시 코드를 추출하고 추출로 인해 항목 추가 처리에 영향을 주지 않음을 확인한다.  
어떻게 보면 의존관계 이삭줍기는 일종의 책임 회피처럼 느껴질 수도 있는데, 일부 동작들은 유지하지만 그 외의 동작들은 무방비 상태로 처리하기 떄문이다.
의존관계 이삭줍기는 중요 동작이 다른 동작들과 뒤섞여 있을 때 특히 효과적이다.

### 메소드 객체 추출
이 기법은 워드 커닝햄이 처음 발표했으며, 조작된 추상화라는 개념의 전형을 보여준다.  
괴물 메소드에서 메소드 객체를 추출한 겨우, 이 클래스의 유일한 책임은 괴물 메소드의 일을 수행하는 것이다.  
메소드 매개변수는 이 신규 클래스 생성자의 매개변수가 되고, 괴물 메소드의 코드는 이 클래스의 run execute메소드 내로 이동된다.  
코드를 신규 클래스로 이동하면 리팩토링이 매우 쉬워진다. 메소드 내의 임시 변수를 인스턴스 변수로 바꿈으로서 메소드를 분할할 때 감지 변수로서 사용할 수도 있다.  
메소드 객체 추출은 매우 과감한 방법이지만, 감지 변수 도입 기법과 달리 여기에 사용되는 변수들은 배포 코드에서도 사용된다. 따라서 테스트 루틴을 그대로 남길 수 있다.

## 전략
### 뼈대 메소드
조건문을 포함하는 코드 내에서 메소드로서 추출할 부분을 찾을 경우, 두 가지 방법 중 하나를 선택할 수 있다.  
조건 부분과 처리 부분을 별도로 추출하는 방법과 함께 추출하는 방법이다. 우선 별도로 추출하는 방법의 예제를 보자.
```Java
if(marginalRate() > 2 && order.hasLimit()) {
  order.readjust(rateCalculator.rateForToday());
  order.recalculate();
}
```
조건 부분과 처리 부분을 두개의 서로 다른 메소드로 추출하면, 나중에 메소드의 로직을 구성하기 쉽다.
```Java
if(orderNeedsRecalculation(order)) {
  recalculateOrder(order, reCalculator);
}
```
이것을 뼈대 메소드라고 부른다. 메소드에 남은것은 뼈대, 즉 제어 구조 및 다른 메소드로의 위임 뿐이기 때문이다.

### 처리 시퀀스 발견
이번에는 함께 추출하는 방법의 예제다.  
조건 부분과 처리 부분을 묶에서 한개의 메소드로 추출하면, 공통의 처리 시퀀스를 찾기 쉽다.
```Java
...
recalcurateOrder(order, rateCalculator);
...

void recalculateOrder(Order order, RateCalculator rateCalculator) {
  if(marginalRate() > 2 && order.hasLimit()) {
    order.readjust(rateCalculator.rateForToday());
    order.recalculate();
  }
}
```
메소드에 남은 부분은 차례로 수행되는 처리 시퀀스에 불과한 것임을 알 수 있다. 처리 순서가 눈에 잘 보인다면 더 명확해진다.  
필자는 실제로 뼈대 메소드와 처리 시퀀스 발견 기법을 오가며 작업한다고 한다.  
제어 구조를 이해한 후에 리팩토링해야 한다고 느껴질때는 우선 뼈대 메소드를 작성한다.  
그리고 포괄적인 처리 시퀀스를 찾음으로서 코드가 명확해진다고 판단되면 처리 시퀀스 발견을 시도한다.  
결국 어떤 전략을 선택할지는 추출 과정에서 설계를 이해한 내용에 따라 달라질 수있다.

### 우선 현재 클래스 내에서 추출
괴물 메소드에서 메소드를 추출할 떄, 추출을 검토중인 코드의 일부가 실제로는 다른 클래스의 것임을 알게 될 때가 있다.  
이 사실은 붙이고자 하는 이름에 잘 나타나는데 어떤 코드 조각을 봤을 때 그 안에서 사용하고 있는 변수 중 하나와 동일한 이름을 붙이고 싶어진다면,  
해당 코드는 아마도 그 변수가 속한 클래스로 옮기는 편이 나을 것이다.  
메소드를 별도의 클래스로 옮기는 것은 어떤 방향으로 변경하고 추출하는 것이 올바른지 명확해진 이후에 진행하는 것이 오류 가능성을 줄일 수 있다.

### 작은 조각 추출
처음에는 작은 코드 조각을 추출해봤자 괴물 메소드에 아무 영향도 미치지 못하는 것처럼 느껴질 수 있다.  
하지만 꾸준히 계속해나가면 기존 메소드를 다른 각도에서 바라볼 수 있게 된다.  
예전에는 잘 보이지 않던 처리 시퀀스를 발견하든가, 좀더 적절한 메소드 정리 방법이 눈에 보이던가 하게 되는데, 이런 방향성이 보이기 시작하면 그쪽으로 나아가면 된다.  
이는 처음부터 대규모 덩어리로 분할하려는 것보다 효과적인데, 처음부터 대규모 덩어리로 분할하는 것은 생각만큼 쉽지 않으며 안전하지도 않다.


### 추출을 다시 할 각오
몇번 정도 추출작업을 해보면, 새로운 기능을 쉽게 수용할 수 있는 더 나은 방법이 발견되곤 하는데, 때로는 이미 했던 추출 작업을 원래대로 되돌리고 다른 방법으로 다시 추출하는것이 최선책일수도 있다.  
그렇다고 이미 수행했던 추출 작업이 모두 헛된것은 아닌데, 기존 설계에 대한 통찰력과 앞으로의 방향성이라는 매우 중요한 것을 배울 수 있기 때문이다.
