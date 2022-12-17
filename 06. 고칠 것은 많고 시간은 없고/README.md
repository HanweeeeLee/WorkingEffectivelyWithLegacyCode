# 06. 고칠 것은 많고 시간은 없고
코드 변경을 위해 의존 관계를 제거하고 테스트 루틴을 작성하는 작업은 개발자의 시간을 많이 빼앗는다. 하지만 대부분의 경우, 결국은 개발 시간과 시행착오를 줄여준다. 그런데 '결국'이란 대체 언제를 가리키는 것일까?  
-> 프로젝트마다 다르다.  
절약되는 시간은 오류를 수정하는 시간 뿐 아니라 오류를 탐지하는 시간도 포함된다. 테스트 루틴 덕분에 문제를 파악하는 일이 훨씬 간단하게 해결될 때가 많다.  
이런 일이 일어나는 시점은 운이 좋다면 빠른 시일 내에, 운이 나쁘면 몇년 후  
  
시간 압박이 주어진 상황에서 테스트 루틴 작성 여부를 판단하는 데 가장 문제가 되는 것은 기능 구현에 얼리는 시간을 알 수 없다는 점이다.  
레거시 코드의 경우, 정밀하게 시간을 추정하는 것은 매우 어려운 일이다. 자세한 설명은 16장에서..  
이런 경우 일단개발, 추후 테스트를 하게되는 경우가 있음. 하지만 실상은 그래놓고 안함~  
딜레마다.  

## 발아 메소드 
### 예시
시스템에 새로운 기능을 추가해야 하는데 이 기능을 완전히 새로운 코드로 표현할 수 있다면, 새로운 메소드로서 이 기능을 구현한 후 이 메소드를 필요한 위치에서 호출하는 방법을 사용할 수 있다. 호출을 수행하는 코드를 테스트 루틴으로 보호하기는 어렵더라도 최소한 새로운 코드에 대한 테스트 루틴은 작성할 수 있다.
```Java
public class TransactionGate {
    public void postEntries(List entries) {
        for (Iterator it = entries.Iterator(); it.hasNext(); ) {
            Entry entry = (Entry)it.next();
            entry.postDate();
        }
        transactionBundle.getListManager().add(entries);
    }
    ...
}
```
각 항목에 날짜를 설정하고 transactionBundle 클래스에 저장하는데, 이 때 신규 항목인지 검사하는 코드를 추가할 필요가 있다.  
반복문이 수행되기 전에 메소드의 첫 부분에 추가해야 한다. 하지만 실제로는 다음코드처럼 반복문 내에서 추가할 수도 있다. 
```Java
public class TransactionGate {
    public void postEntries(List entries) {
        List entriesToAdd = new LinkedList();
        for (Iterator it = entries.Iterator(); it.hasNext(); ) {
            Entry entry = (Entry)it.next();
            if (!transactionBundle.getListManager().hasEntry(entry)) {
                entry.postDate();
                entriesToAdd.add(entry);
            }
        }
        // transactionBundle.getListManager().add(entries);
        transactionBundle.getListManager().add(entriesToAdd);
    }
    ...
}
```
날짜 설정과 중복 항목 검사라는 두 개의 동작이 섞여있기 때문에 코드가 안좋다. 이 메소드는 이해하기가 어려워졌다.  
또 임시 변수를 새로 도입했는데, 임시변수가 반드시 나쁘다고는 할 수 없지만 새로운 코드를 불러들이기가 쉽다.  
이제 테스트 주도 개발에 의해 uniqueEntries 메소드를 새롭게 작성해보자.

```Java
public class TransactionGate {
    ...
    List uniqueEntries(List entries) { // 윗부분에서 추출해 함수를 만든듯
        List result = newArrayList();
        for (Iterator it = entries.Iterator(); it.hasNext(); ) {
            Entry entry = (Entry)it.next();
            if (!transactionBundle.getListManager().hasEntry(entry)) {
                result.add(entry);
            }
        }
        return result;
    }
    ...
}
```
이 메소드의 테스트 루틴은 간단히 작성할 수 있다. 메소드를 작성한 후, 기존 코드에 이 메소드의 호출을 추가한다. 
```Java
public class TransactionGate { // 맨 위에있던 걸 다시 수정해보자
    public void postEntries(List entries) {
        List entriesToAdd = uniqueEntries(entries); // 아까 만들어놨던거 호출
        for (Iterator it = entries.Iterator(); it.hasNext(); ) {
            Entry entry = (Entry)it.next();
            entry.postDate();
        }
        // transactionBundle.getListManager().add(entries);
        transactionBundle.getListManager().add(entriesToAdd);
    }
    ...
}
```
여전히 임시 변수가 남아있지만 코드는 한결 정리됐다.  
지금까지 설명한 것은 발아 메소드(sprout method)의 예다. 
### 발아 메소드의 작성 순서 
1. 어느 부분에 코드 변경이 필요한지 식별한다.
2. 메소드 내의 특정 위치에서 일련을 명령문으로서 구현할 수 있는 변경이라면, 필요한 처리를 수행하는 신규 메소드를 호출하는 코드를 작성한 후 주석처리 한다.
3. 호출되는 메소드가 필요로 하는 지역 변수를 확인하고, 이 변수들을 신규 메소드 호출의 인수로 전달한다.
4. 호출하는 메소드에 값을 반환해야 하는지 여부를 결정한다. 값을 반환해야 한다면, 반환 값을 변수에 대입하도록 호출 코드를 변경한다.
5. 새롭게 추가되는 메소드를 테스트 주도 개발 방법을 사용해 작성한다.
6. 앞서 주석 처리했던 신규 메소드 호출 코드의 주석을 제거한다. 
  
  
독립된 한 개의 기능으로서 코드를 추가하는 겨웅나 메소드의 테스트 루틴이 아직 준비되지 않은 경우에는 발아 메소드의 사용을 권장한다. 코드를 인라인 형태로 추가하는 것보다 훨씬 바람직한 결과로 이어지기 때문이다.  

### 장점과 단점
단점: 이 메소드를 사용하는 것은 원래의 메소드와 클래스를 잠시 포기하는것과 같다. 원래의 메소드는 테스트 루틴으로 보호하는 것도 아니고, 개선하는 것도 아니다. 그저 신규 메소드로서 새로운 기능을 추가하는 것이다.  
기존의 메소드는 상당량의 복잡한 코드와 한 개의 신규 발아 메소드가 포함되는 형태가 될 수 있는데, 이처럼 일부 위치에 대해서만 작업하면 코드의 의도를 이해하기 힘들어지기 때문에 기존 메소드는 만들다 만 것 같은 상태가 되어버린다.  
이는 적어도 원래의 클래스를 나중에 테스트 루틴으로 보호할 떄 추가적인 작업을 해야 한다는것을 의미한다.  

장점: 기존 코드와 새로운 코드를 확실히 구분 할 수 있다. 기존 코드를 테스트 루틴으로 보호할 수 없더라도 최소한 변경 부분을 개별적으로 이해하고 고칠 수 있으며, 기존 코드 사이의 인터페이스도 분명해진다. 또한 영향을 받는 변수들을 모두 파악할 수 있기 떄문에 코드의 정확성도 쉽게 판단할 수 있다. 

## 발아 클래스
클래스 단위의 테스트가 의존성 등의 이유로 (주어진 시간 안에) 테스트 하네스 안으로 넣지 못한다면 이 방법을 사용해보자.  

```Cpp
std::string QuarterlyReportGenerator::generate() {
    std::vector<Result> results = database.queryResults(beginDate, endDate);
    std::string pageText;
    pageText += "<html><head><title>"
        "quarterly Report"
        "</title></head><body><table>";
    if (results.size() != 0) {
        for (std::vector<Result>::iterator it = results.begin(); it != results.end() ; ++it) {
            pageText += "<tr>";
            pageText += "<td>" + it->department + "</td>";
            pageText += "<td>" + it->manager + "</td>";
            char buffer [128];
            sprintf(buffer, "<td>$%d</td>", it->netProfit / 100);
            pageText += std::string(buffer);
            sprintf(buffer, "<td>$%d</td>", it->operatingExpense / 100);
            pageText += std::string(buffer);
            pageText += "</tr>";
        }
    } else {
        pageText += "No results for this period";
    }
    pageText += "</table>";
    pageText += "</body>";
    pageText += "</html>";
    return pageText;
}
```
이 코드가 생성하는 HTML 테이블에 헤더 행을 추가한다고 가정하자. 헤더 행은 다음과 같다.
```HTML
"<tr><td>Department</td><td>Manager</td><td>Profit</td><td>Expenses</td></tr>"
```
또한 이 클래스는 무척 커서 클래스 전체를 테스트 하네스에 추가하는 데만 꼬박 하루가 걸리고, 현재 그럴 여유가 없다고 가정하자.  
이 변경은 QuarterlyReportTableHeaderProducer라는 소규모 클래스로 구현할 수 있다.  
테스트 주도 개발을 통해 클래스를 개발해보자.  
```Cpp
using namespace std;
class QuarterlyReportTableHeaderProducer {
public: 
    string makeHeader();
};

string QuarterlyReportTableHeaderProducer::makeHeader() {
    return "<tr><td>Department</td><td>Manager</td><td>Profit</td><td>Expenses</td></tr>"
}
```
QuarterlyReportGenerator::generate() 메소드 내에서 이 클래스를 인스턴스화해 직접 호출한다.
```Cpp
...
QuarterlyReportTableHeaderProducer producer;
pageText += producer.makeHeader();
...
```
이런 질문을 할 수 있다.  
"이런거 하려고 클래스까지 만드나요? 설계상 전혀 도움이 되지 않습니다."  
맞는 말이다. 이 방법을 사용한 유일한 이유는 의존 관계의 복잡성에서 벗어나기 위한 것이다.  
  
이 클래스를 QuarterlyReportTableHeaderGenerator라 명명하고, 다음과 같은 인터페이스를 갖게 하면 어떻게 될까?
```Cpp
class QuarterlyReportTableHeaderGenerator {
public:
    string generate();
};
```
QuarterlyReportTableHeaderGenerator 클래스는 QuarterlyReportGenerator와 마찬가지로 HTML을 생성한다.  
둘 다 문자열 값을 반환하는 generate()메소드를 포함하며, 인터페이스 클래스를 정의한 후 이 클래스들에 상속함으로써 공통의 코드를 활용할 수 있다.  
```Cpp
class HTMLGenerator {
public:
    virtual ~HTMLGenerator() = 0;
    virtual string generate() = 0;
};

class QuarterlyReportTableHeaderGenerator : public HTMLGenerator {
public:
    ...
    virtual string generate();
    ... 
};

class QuarterlyReportGenerator : public HTMLGenerator {
public:
    ...
    virtual string generate();
    ... 
};
```
작업을 하다보면, QuarterlyReportGenerator는 테스트 루틴으로 보호하고 대부분의 처리는 HTML 생성 클래스들이 수행하도록 코드르 변경할 수도 있을 것이다.  
발아 클래스를 애플리케이션의 기존 메커니즘에 도저히 집어넣을 수 없는 경우에는 발아 클래스가 새로운 메커니즘이 된다.  
발아 클래스를 작성하는 경우는 본질적으로 다음 두 가지다.  
1. 어떤 클래스에 완전히 새로운 역할을 추가하고 싶은 경우
2. 기존 클래스에 약간 기능을 추가하고 싶지만 그 클래스를 테스트 하네스 내에서 테스트할 수 없는 경우

### 발아 클래스의 작성 순서 
1. 어느 부분의 코드를 변경해야 하는지 식별한다.
2. 메소드 내의 특정 위치에서 일련의 명령문으로 변경을 구현할 수 있다면, 변경을 구현할 크래스에 적합한 이름을 생각한다. 이어서 해당 위치에 그 클래스의 객체를 생성하는 코드를 삽입하고, 클래스 내의 메소드를 호출하는 코드를 작성한다. 그리고 이 코드를 주석처리한다.
3. 호출 메소드의 지역 변수 중에 필요한 것을 결정하고, 이 변수들을 클래스의 생성자가 호출될 떄의 인수로 만든다.
4. 발아 클래스가 호출 메소드에 결과 값을 반환해야 하는지 판단한다. 값을 반환해야 한다면 그 값을 제공할 메소드를 클래스에 추가하고, 이 메소드를 호출해 반환 값을 받아오는 코드를 호출 메소드에 추가한다.
5. 새로운 클래스를 테스트 주도 개발로 작성한다.
6. 앞서 주석 처리했던 주석을 제거하고, 객체 생성과 호출을 활성화한다.

### 장점과 단점
장점: 코드를 직접 재작성하는 경우보다 확신을 갖고 변경 작업을 진행할 수 있다.  
단점: 메커니즘이 복잡하다. 추상적인 처리 부분과 다른 클래스 내의 처리부분으로 이뤄지기 때문이 이해하기가 어렵다.  


## 포장 메소드
기존 메소드에 동작을 추가하는 것은 간단한 일이지만, 이것이 옳지 않은 접근법일 때가 많다. 추가되는 메소드 들은 의심스러운 것들이다. 개발자가 그런 코드를 추가하는 이유는 단지 추가 코드가 기존 코드와 동시에 실행되기 때문일 때가 있는데, 이는 과거에 일시적 결합(temporal coupling)이라 불리던 현상으로서 과도하게 사용하면 코드의 품질을 저하시킨다.  
동작을 추가할 때 복잡하지 않은 기법을 사용할 필요가 있다. 포장 메소드를 사용해보자.  
### 예시 
```Cpp
public class Employee {
    ...
    public void pay() {
        Money amount = new Money();
        for (Iterator it = timecards.iterator(); it.hasNext(); ) {
            Timecard card = (TimeCard)it.next();
            if (payPeriod.contains(date)) {
                amount.add(card.getHours() * payRate);
            }
        }
        payDispatcher.pay(this, date, amount);
    }
}
```
이 메소드는 직원의 타임카드(근무시간기록표)를 집계하고 급여 정보를 PayDispatcher객체로 보낸다.  
여기서 직원에게 급여를 지급할 때 직원 이름으로 파일을 갱신해 별도의 보고서 작성 소프트웨어로 보내야 한다면?  
코드를 추가하기 가장 쉬운 곳은 pay 메소드다. 결국 파일 처리는 pay 메소드와 동시에 일어나야 하기 때문이다. 이를 코드로 구현하면?  
```Cpp
public class Employee {
    ...
    // public void pay() {
    private void dispatchPayment() {
        Money amount = new Money();
        for (Iterator it = timecards.iterator(); it.hasNext(); ) {
            Timecard card = (TimeCard)it.next();
            if (payPeriod.contains(date)) {
                amount.add(card.getHours() * payRate);
            }
        }
        payDispatcher.pay(this, date, amount);
    }

    public void pay() {
        logPayment();
        dispatchPayment();
    }

    private void logPayment() {
        ...
    }
    ...
}
```
이렇게 변경되면 이 클래스를 사용하는 고객은 이러하면 변경을 알지 못해도 된다.  
기존 메소드와 이름이 같은 메소드를 새로 생성하고 기존 코드에 처리를 위임한다. 이것이 포장메소드  
  
다음은 포장 메소드의 다른 형태이다. 아직 호출된 적이 없는 새로운 메소드를 추가하고자 할 때 사용할 수 있다.

```Cpp
public class Employee {
    public void makeLoggedPayment() {
        logPayment();
        pay();
    }

    public void pay() {
        ...
    }

    private void logPayment() {
        ...
    }
}
```
이제 Employee 클래스 사용자는 어느 방법으로 지불을 수행할지 선택할 수 있다.  

#### 단점
1. 새로 추가하려는 기능의 로직이 기존 기능의 로직과 통합될 수 없다. 새로운 기능은 기존 기능의 이전이나 이후에 수행돼야 한다. 하지만 이는 반드시 부정적이지 않으며, 가능하다면 이 기법을 사용하는 것이 좋다.  
2. 기존 메소드 내의 코드를 위해 새로운 이름을 고안해야 한다는 점. 위 예제에서는 pay() 메소드 내의 코드를 dispatchPayment()라고 명명했다. 사실 이름이 적절하지 않음
```Cpp
public void pay() {
    logPayment();
    Money amount = calculatePay();
    dispatchPayment(amount);
}
```
> 사실 이렇게 가야 하는듯

### 포장 메소드 작성 순서
1. 변경해야 할 메소드를 식별한다.
2. 변경이 메소드 내의 특정 위치에서 일련의 명령문으로 구현 가능하다면, 메소드 이름을 바꾸고 기존 메소드와 동일한 이름과 서명을 갖는 메소드를 새로 작성한다. 이때 서명을 그대로 유지하는 것을 잊지 말자.
3. 새로운 메소드에서 기존 메소드를 호출하도록 한다.
4. 새로운 기능을 위한 메소드를 테스트 주도 개발을 통해 작성하고, 이 메소드를 단계 2에서 작성한 신규 메소드에서 호출한다.

#### 포장 메소드의 두 번째 형태, 즉 기존 메소드와 동일한 이름을 사용하지 않아도 되는 경우
1. 변경하고자 하는 메소드를 식별한다.
2. 변경이 메소드 내의 특정 위치에서 일련의 명령문으로 구현 가능하다면, 변경을 구현할 메소드를 테스트 주도 개발에 의해 새로 작성한다.
3. 새 메소드와 기존 메소드를 호출하는 별도의 메소드를 작성한다.

### 장점과 단점
#### 장점
1. 발아 메소드나 발아 클래스는 기존 메소드에 코드가 추가되기 때문에 코드의 길이가 최소한 한 줄이라도 늘어나지만, 포장 메소드는 기존 메소드의 길이가 변하지 않는다.
2. 신규 기능이 기존 기능과 분명히 독립적으로 만들어진다.
#### 단점
1. 다소 부적절한 이름을 붙이기 쉽다.












































