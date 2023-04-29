# 25. 의존 관계 제거 기법 part2

## 정적 set 메소드 도입
값이 바뀔 수 있는 전역 데이터를 사용하면 테스트 조건에 맞도록 확인해야만 테스트를 할 수 있다.  

테스트하기위한 코드를 작성해보자.

### 예제1
```cpp
void MessageRouter::route(Message *message) {
    ...
    Dispatcher *dispatcher = ExternalRouter::instance()->getDispatcher();
    if (dispatcher != NULL)
        dispatcher->sendMessage(message);
}
```
MessageRouter 클래스는 발송자들을 얻기 위해 여러 곳에서 싱글톤을 사용한다.  
ExternalRouter클래스는 발송자를 위한 하나의 get메소드를 가진다. 발송자에 서비스를 제공하는 외부 라우터를 대체함으로써 그 발송자를 다른 발송자로 바꿀 수 있다.  
다음은 정적 set 메소드를 도입하기 이전의 ExtenralRouter클래스임.
```cpp
class ExternalRouter {
private:
    static ExternalRouter *_instance;
public:
    static ExternalRouter *instance();
    ...
};

ExternalRouter *ExternalRouter::_instance = 0;
ExternalRouter *ExternalRouter::instance() {
    if (_instance == 0) {
        _instance = new ExternalRouter;
    }
    return _instance;
}
```
> instance메소드를 처음 호출할 떄 생성된다. 다른 라우터로 대체하려면 instance의 반환 값을 변경해야 한다.
  
개선해보자.
```cpp
void ExternalRouter::setTestingInstance(ExternalRouter *newInstance) {
    delete _instance;
    _instance = newInstance;
}
```
> 다른곳에 구멍 뚫음
```cpp
class TestingExternalRouter: public ExternalRouter {
public:
    virtual void Dispatcher *getDispatcher() const {
        return new FakeDispatcher;
    }
}
```

### 예제2
```Java
public class RouterFactory {
    static Router makeRouter() {
        return new EWNRouter();
    }
}
```
> 전역 팩토리. 
```Java
interface RouterServer {
    Router makeRouter();
}

public class RouterFactory implements RouterServer {

    static Router makeRouter() {
        return server.makeRouter();
    }

    static setServer(RouterServer server) {
        this.server = server;
    }

    static RouterServer server = new RouterServer() {
        public RouterServer makeRouter() {
            return new EWNRouter();
        }
    };

}
```

테스트 루틴 안에서는 다음과 같이 하면 됨
```Java
protected void setup() {
    RouterServer.setServer(new RouterServer() {
        public RouterServer makeRouter() {
            return new FakeRouter();
        }
    });
}
```

### 단계
1. 싱글톤 클래스 생성자의 보호 수준을 낮춰서 서브클래스에 의한 가짜 클래스 작성을 가능하게 한다.
2. 싱글톤 클래스에 정적 set 메소드를 추가한다. set메소드에 싱글톤 클래스에 대한 참조가 전달되도록 한다. 또한 set메소드가 새로운 객체를 설정하기 전에 싱글톤 객체를 확실히 제거한다.
3. 테스트 설정을 위해 싱글톤 내의 private이나 protected 메소드에 접근해야 한다면, 싱글톤을 서브클래스화하거나 인스턴스를 추출해 그 인스턴스를 구현한 객체 잠조를 싱글톤에서 유지하도록 한다.
  

  
## 연결 대체
이 파트 자바, 스위프트를 쓰는 우리랑은 무관한 파트같아서 패스..;;  
인터페이스와 상속을 못쓰는 C는 어떻게 저렇게 개발하나 에 대한 이야기 인듯

## 생성자 매개변수화
생성자 안에서 하나의 객체를 생성하는 경우, 가장 쉽게 대체하는 방법은 생성을 외부에서 하는것임.
```Java
public class MailChecker {
    public MailChecker(int checkPeriodSeconds) {
        this.receiver = new MailReceiver();
        this.checkPeriodSeconds = checkPeriodSeconds;
    }
    ...
}
```
->
```Java
public class MailChecker {
    public MailChecker(int checkPeriodSeconds, MailReceiver receiver) {
        // this.receiver = new MailReceiver();
        this.receiver = receiver;
        this.checkPeriodSeconds = checkPeriodSeconds;
    }
    ...
}
```
-> 
```Java
public class MailChecker {

    public MailChecker(int checkPeriodSeconds) {
        this(new MailReceiver(), checkPeriodSeconds);
    }

    public MailChecker(int checkPeriodSeconds, MailReceiver receiver) {
        // this.receiver = new MailReceiver();
        this.receiver = receiver;
        this.checkPeriodSeconds = checkPeriodSeconds;
    }
    ...
}
```
> 이렇게 기존꺼 수정안하게 서명 유지 가능

### 단계
1. 매개변수화하길 원하는 생성자를 식별해내고 그것을 복사해둔다.
2. 그 객체의 생성자에 하나의 매개변수를 추가한다. 당신은 이 객체 생성을 대체할 것이다. 객체 생성을 제거하고, 매개변수로 얻은 할당 값을 해당 객체를 위한 인스턴스 변수에 추가한다.
3. 사용하는 언어에 있는 생성자로부터 생성자를 호출할 수 있다면, 옜 생성자의 본문을 제거한 후 그것을 옜 생성자에 대한 호출로 대체한다. 옜 생성자 내에 있는 새로운 생성자에 대한 호출에 새로운 표현을 추가한다. 다른 생성자로부터 생성자를 호출할 수 없다면 생성자들 간에 중복된 것을 추출해 새로운 메소드로 만들어야 할지도 모른다.

## 메소드 매개변수화
객체를 내부적으로 생성하는 하나의 메소드가 있고, 이를 감지하거나 분리하기 위해 이 객체를 대체하고자 한다.
```cpp
void TestCase::run() {
    delete m_result;
    m_result = new TestResult;
    try {
        setUp();
        renTest(m_result);
    }
    catch (exception& e) {
        result->addFailure(e, this);
    }
    tearDown();
}
```
->
```cpp
void TestCase::run(TestResult *result) {
    delete m_result;
    // m_result = new TestResult;
    m_result = result;
    try {
        setUp();
        renTest(m_result);
    }
    catch (exception& e) {
        result->addFailure(e, this);
    }
    tearDown();
}
```
### 단계
메소드 매개변수화를 사용하려면 다음에 소개하는 단계들을 밟는다.
1. 대체하길 원하는 메소드를 식별해내고 그것을 복사해둔다.
2. 그 객체의 생성자에 하나의 매개변수를 추가한다. 당신은 이 객체 생성을 대체할 것이다. 객체 생성을 제거하고 매개변수로 얻은 할당 값을 해당 객체를 위한 인스턴스 변수에 추가한다.
3. 복사된 메소드의 본문을 지우고 매개변수화된 메소드를 호출하도록 만든다. 이때 기존 객체에서 사용된 객체 생성 표현식을 사용한다.

## 매개변수 원시화
의존성관계가 너무 개판이어서 테스트를 하려면 작업량이 너무 많을 때 사용.  

```Cpp
TEST(hasGapFor, Sequence) {
    vector<unsigned int> baseSequence;
    baseSequence.push_back(1);
    baseSequence.push_back(0);
    baseSequence.push_back(0);
    vector<unsigned int> pattern;
    pattern.push_back(1);
    pattern.push_back(2);
    CHECK(SequenceHasGapFor(baseSequence, pattern));
}
```

테스트 하네스 안에서 SequenceHasGapFor를 위한 기능을 만든다면, 새 기능을 위임하는 조금 단순한 함수를 Sequence클래스상에 작성할 수 있다.
```Cpp
bool Sequnce::hasGapFor(Sequence& pattern) const {
    vector<unsigned int> baseRepresentation = getDurationsCopy();
    vector<unsigned int> patternRepresentation = pattern.getDurationsCopy();
    return SequenceHasGapFor(baseRepresentation, patternRepresentation);
}
```
이 함수는 duration에 대한 배열을 얻기 위해 또 다른 함수 하나가 필요해진다. 그래서 다음과 같은 함수 하나를 작성한다.
```Cpp
vector<unsigned int> Sequence::getDurationsCopy() const {
    vector<unsigned int> result;
    for (vector<Event>::iterator it = events.begin() ; it != events.end() ; ++it) {
        result.push_back(it->duration);
    }
    return result;
}
```

우리가 해왔던 끔찍한 일들을 나열해보자.
1. Sequence 클래스 내부 표현을 드러냈다.
2. Sequence 클래스 안에 있는 함수 가운데 몇 가지를 자유 함수로 밀어냄으로써 Sequence의 구현을 좀 더 이해하기 어렵게 만들었다.
3. 테스트 루틴 없는 코드를 작성했다.
4. 시스템에 있는 자료를 복제했다.
5. 문제를 질질 끌었다. 도메인 클래스들과 인프라스트럭처 간의 의존 관계를 제거하는 일을 열심히 하기는 커녕 시작하지도 않았다.
  
이러한 모든 담점들에도 불구하고 우리는 테스트 루틴을 추가할 수 있었다. 

### 단계
1. 클래스상에서 해야 하는 작업을 수행하는 자유 함수를 하나 만든다. 이 과정에서 작업 수행에 필요한 중간 표현을 개발한다.
2. 해당 표현을 만들고 그 표현을 새 함수에 위임하는 함수를 클래스에 추가한다.

## 특징 끌어올리기
클래스 상에 있는 메소드 클러스터, 그리고 그 클러스터와는 상관없지만 해당 클래스를 인스턴스화하지 못하게 방해하는 의존관계가 있는 가운데 작업해야 할 때가 있다.  

```Java
public class Scheduler {
    private List items;
    public void updateScheduleItem(ScheduleItem item)
        throws SchedulingException {
            try {
                validate(item);
            }
            catch (ConflictException e) {
                throw new SchedulingException(e);
            }
            ...
        }
    ...

    private void validate(ScheduleItem item) {
        throws ConflictException {
            // DB호출하기 
            ...
        }
    }

    public int getDeadtime() {
        int result = 0;
        for (Iterator it = items.iterator() ; it.hasNext() ; ) {
            ScheduleItem item = (ScheduleItem)it.next();
            if (item.getType() != ScheduleItem.TRANSIENT && notShared(item)) {
                result += item.getSetupTime() + clockTime();
            }
            if (item.getType() != ScheduleItem.TRANSIENT) {
                result += item.finishingTime();
            } else {
                result += getStandardFinish(item);
            }
        }
        return result;
    }
}
```
관심 있는 메소드를 끌어올려 슈퍼클래스 안에 두자. 이렇게 함으로써 나쁜 의존 관계들을 이 슈퍼클래스 안에 둘 수 있고, 그 나쁜 의존 관계들은 더 이상 테스트 루틴을 방해하지 않는다.
```Java
// public class Scheduler {
public class Scheduler extends SchedulingServices {
    private List items;
    public void updateScheduleItem(ScheduleItem item)
        throws SchedulingException {
            try {
                validate(item);
            }
            catch (ConflictException e) {
                throw new SchedulingException(e);
            }
            ...
        }
    ...

    private void validate(ScheduleItem item) {
        throws ConflictException {
            // DB호출하기 
            ...
        }
    }

    // public int getDeadtime() { 테스트 하고자 하는 특징
    //     int result = 0;
    //     for (Iterator it = items.iterator() ; it.hasNext() ; ) {
    //         ScheduleItem item = (ScheduleItem)it.next();
    //         if (item.getType() != ScheduleItem.TRANSIENT && notShared(item)) {
    //             result += item.getSetupTime() + clockTime();
    //         }
    //         if (item.getType() != ScheduleItem.TRANSIENT) {
    //             result += item.finishingTime();
    //         } else {
    //             result += getStandardFinish(item);
    //         }
    //     }
    //     return result;
    // }
}

public abstract class SchedulingServices { // 추가

    protected List items;

    protected boolean notShared(ScheduleItem item) {
        ...
    }

    protected int getClockTime() {
        ...
    }

    protected int getStandardFinish(ScheduleItem item) {
        ...
    }

    public int getDeadtime() { 테스트 하고자 하는 특징
        int result = 0;
        for (Iterator it = items.iterator() ; it.hasNext() ; ) {
            ScheduleItem item = (ScheduleItem)it.next();
            if (item.getType() != ScheduleItem.TRANSIENT && notShared(item)) {
                result += item.getSetupTime() + clockTime();
            }
            if (item.getType() != ScheduleItem.TRANSIENT) {
                result += item.finishingTime();
            } else {
                result += getStandardFinish(item);
            }
        }
        return result;
    }
    ...
}
```

이제 우리는 테스트 서브클래스를 만들 수 있다. 

### 단계
1. 끌어올리고자 하는 메소드들을 식별한다.
2. 해당 메소드들을 포함하는 클래스를 위한 하나의 추상 슈퍼클래스를 생성한다.
3. 해당 메소드들은 슈퍼클래스에 복사한 후 컴파일한다.
4. 새로운 슈퍼클래스에 대해 컴파일러 경고를 고려함으로써 빠진 참조를 모두 복사할 수 있도록 한다. 이 작업을 하는 동안 오류 가능성을 줄이기 위해 서명 유지를 사용하는것을 잊지 말자.
5. 두 클래스가 성공적으로 컴파일되고 나면 추상 클래스를 위한 하나의 서브클래스를 생성하고, 그 클래스를 테스트 루틴 안에서 설정할 수 있게 하는 메소드들을 추가한다.


