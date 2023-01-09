# 09. 뚝딱! 테스트 하네스에 클래스 제대로 넣기
클래스를 테스트 하네스에 넣는 작업은 어려운일이다. 다음은 가장 일반적으로 직면하는 네 가지 문제점이다.
 1. 클래스 객체를 쉽게 생성할 수 없다.
 2. 클래스를 포함하는 테스트 하네스를 쉽게 빌드할 수 없다.
 3. 반드시 사용해야 하는 생성자가 부작용을 일으킨다.
 4. 생성자의 내부에서 상당량의 처리가 일어나며, 그 내용을 알아내야만 한다.

## 성가신 매개변수
신용카드 청구 시스템을 구현한 코드에 테스트 루틴을 포함하지 않는 CreditValidator라는 이름의 자바 클래스가 있다. 
```Java
public class CreditValidator {
    public CreditValidator(RGHConnection connection, CreditMaster: master, String validatorID) {
        ...
    }

    Certificate validatorCustomer(Customer customer) throws InvalidCredit {
        ...
    }
    ...
}
```
이 클래스는 몇 가지 책임을 가지고 있는데, 그중 하나는 고객의 신용잔고가 유효한지 검증하는 것이다.  
지금 이 클래스에 새로운 메소드를 추가해야만 한다고 하자. getValidationpercent라는 메소드인데 CreditValidator 객체가 존재하는 동안에 validateCustomer 메소드 호출이 성공했던 비율을 알려주는 역할을 한다.  
테스트 하네스에서 객체를 생성해보자.
```Java
public void testCreate() {
    CreditValidator validator = new CreditValidator(); // 생성자가 없다는 오류가 날것이다.
}
```
RGHConnection, CreditMaster, validatorID 가 필요하다는걸 알았다.

```Java
public class RGHConnection {
    public RGHConnection(int port, String name, string password) thorws IOException {
        ...
    }
}

public class CreditMaster {
    public CreditMaster(String fileName, boolean, isLocal) {
        ...
    }
}
```
RGHConnection 객체는 생성된 후 서버에 연결한다. 이를 통해 고객의 신용을 검증하는데 필요한 정보를 서버로부터 받는다.  
  
CreditMaster는 고객의 신용잔고 결정에 사용되는 몇 가지 정책 정보를 제공한다. 이 클래스의 객체가 생성될 때 파일로부터 정보를 읽어온 후 메모리에 보관한다. 
  
테스트 하네스를 계속 작성해보자.
```Java
public void testCreate() {
    RGHConnection connection = new RGHConnection(DEFAULT_PORT, "admin", "rii8ii9s");
    CreditMaster master = new CreditMaster("crm2.mas", true);
    // CreditValidator validator = new CreditValidator();
    CreditValidator validator = new CreditValidator(connection, master, "a");
}
```
이것을 사용할 수 있을까?  
테스트 단계에서 RGHConnection 객체가 서버와 실제로 연결되는 것은 좋은 생각이 아니다. 서버와의 연결이 오래걸리수도 있고 24시간 동작하는 서버가 아닐수도 있다.  
반면 CreditMaster클래스는 특별히 문제될것은 없다.  
따라서 testCreate를 작성하는데 있어서 문제가 되는것은 RGHConnection이다. 가짜 RGHConnection 객체를 생성한 후 CreditValidator가 이 가짜 객체를 진짜 객체로 믿게하면, 서버연결없이 테스트 할 수 있다.  
RGHConnection이 제공하는 메소드를 살펴보자
 | RGHConnection |
 |:---:|
 | + RGHConnection(port, name, password) |
 | + connect() |
 | + disconnect() |
 | + RFDIReportFor(id: int) : RFDIReport |
 | + ACTIOReportFor(customerID : int) ACTIOReport |
 | - retry() |
 | - formPacket() : RFPacket |
  
이번 예제의 경우, 가짜 객체를 만드는 가장 좋은 방법은 RGHConnection 클래스에 인터페이스 추출 기법을 사용하는것이다.  
```Java
public class FakeConnection implements IRGHConnection {
    public RFDIReport report;
    public void connect() {}
    public void disconnect() {}
    public RFDIReport RFDIReportFor(int id) { return report; }
    public ACTIOReport ACTIOReportFor(int customerID) { return null; }
}
```
이 클래스로 다음과 같이 테스트 루틴을 만들 수 있다.
```Java
void testNoSuccess() thorws Exception {
    CreditMaster master = new CreditMaster("crm2.mas", true);
    IRGHConnection connection = new FakeConnection();
    CreditValidator validator = new CreditValidator(connection, master, "a");
    connection.report = new RFDIReport(...);
    Certificate ressult = validator.validatorCustomer(new Customer(...));
    assertEquals(Certificate.VALID, result.getStatus());
}
```
![KakaoTalk_Photo_2022-12-29-14-11-22](https://user-images.githubusercontent.com/60125719/209906245-478ed0f5-3c56-46dd-8832-e9b36104b4f2.jpeg)

이제 CreditValidator 객체를 생성할 수 있으므로 getValidationPercent 메소드를 작성할 수 있다.  
테스트코드를 작성해보자.
```Java
void testAllPassed100Percent() thorws Exception {
    CreditMaster master = new CreditMaster("crm2.mas", true);
    IRGHConnection connection = new FakeConnection("admin", "rii8ii9s");
    CreditValidator validator = new CreditValidator(connection, master, "a");
    connection.report = new RFDIReport(...);
    Certificate ressult = validator.validatorCustomer(new Customer(...));
    assertEquals(100.0, validator.getValidationPercent(), THRESHOLD);
}
```
> 이 테스트 루틴은 한 개의 유효한 신용 증명서를 받았을 떄 검증 가능한 비율이 100%인지 검사한다.  

살펴보니 getValidationPercent는 CreditMaster를 전혀 사용하지 않는다. 그렇다면 CreditMaster를 CreditValidator에게 전달할 필요가 있을까? 없다.  
따라서 테스트 루틴 내에서 다음과 같이 CreditValidator객체를 생성할 수 있다.
```Java
CreditValidator validator = new CreditValidator(connection, null, "a");
```
> null 을 매개변수에 넣는게 좀 어이없을 수 있는데 테스트에선 허용한다.  

## 숨겨진 의존관계
언뜻 보기에 별문제가 없는 클래스가 있다고 하자. 그런데 이 클래스의 생성자를 호출했더니 곧바로 장애물에 부딪혀서 놀라는 경우가 있다. 예를들면 의존관계  
다음은 메일링 리스트를 관리하는 설계가 엉성한 C++ 클래스다.
```Cpp
class mailing_list_dispatcher {
public: 
    mailing_list_dispatcher();
    virtual ~mailing_list_dispatcher;

    void send_message(const std::string& message);
    void add_recipient(const mail_txm_id id, const mail_address& address);
    ...
private:
    mail_service *service;
    int status;
}
```
다음은 이 클래스 생성자의 일부분이다.  
```Cpp
mailing_list_dispatcher::mailing_list_dispatcher() : service(new mail_service), status(MAIL_OKAY) {
    const int client_type = 12;
    service->connect();
    if (service->get_status() == MS_AVAILABLE) {
        service->register(this, client_type, MARK_MESSAGES_OFF);
        service->set_param(client_type, ML_NOBOUNCE | ML_REPEATOFF);
    } else {
        status = MAIL_OFFLINE;
    }
    ...
}
```
생성자 초기화 목록에서 new를 사용해 mail_service 객체를 할당하고 있는데, 이는 좋은 방법이 아니다.  
생성자는 mail_service에 대해 몇 가지 작업을 수행하는 것 외에 의미를 알 수 없는 숫자 12도 사용하고 있다.  
테스트 루틴 내에서 이 클래스의 인스턴스를 생성을 할 수는 있지만 메일 라이브러리 연결, 등록처리를 위한 메일 시스템 설정등을 해야해서 유용하지는 못하다.  
게다가 테스트 중에 send_message 함수를 실행하면 실제로 메일이 날라간다..  
여기서 근본적인 문제는 mail_service에 대한 의존 관계가 mailing_list_dispatcher의 생성자 내부에 숨어있다는 점이다. 가짜 mail_service를 생성하자.  
```Cpp
// mailling_list_dispatcher::mailing_list_dispatcher() : service(new mail_service), status(MAIL_OKAY) {
mailing_list_dispatcher::mailing_list_dispatcher(mail_service *service) : status(MAIL_OKAY) {
    const int client_type = 12;
    service->connect();
    if (service->get_status() == MS_AVAILABLE) {
        service->register(this, client_type, MARK_MESSAGES_OFF);
        service->set_param(client_type, ML_NOBOUNCE | ML_REPEATOFF);
    } else {
        status = MAIL_OFFLINE;
    }
    ...
}
```
그 전과 달라진 점은 mail_service객체를 클래스 외부에서 생성한 후 전달하는 것 뿐이지만 문제를 해결 할 수 있었다.  
이렇게 바꾸면 이 클래스를 사용하는 모든 부분을 다 수정해야 할까?
```Cpp
mailing_list_dispatcher::mailing_list_dispatcher(mail_service *service) : status(MAIL_OKAY) {
    const int client_type = 12;
    service->connect();
    if (service->get_status() == MS_AVAILABLE) {
        service->register(this, client_type, MARK_MESSAGES_OFF);
        service->set_param(client_type, ML_NOBOUNCE | ML_REPEATOFF);
    } else {
        status = MAIL_OFFLINE;
    }
    ...
}

mailing_list_dispatcher::mailing_list_dispatcher::mailing_list_dispatcher(mail_service *service) {
    initialize(service);
}
```
이렇게 하면 됨  

## 복잡한 생성자
생성자 내부에서 많은 수의 객체가 생성되거나 많은 수의 전역 변수에 접근하는 경우, 매개변수 목록의 크기가 지나치게 커질 수 있다.  
```Cpp
class WatercolorPane {
public:
    WatercolorPane(Form *border, WashBrush *brush, Pattern *backdrop) {
        ...
        anteriorPanel = new Panel(border);
        anteriorPanel->setBorderColor(brush->getForeColor());
        backgroundPanel = new Panel(border, backdrop);
        cursor = new FocusWidget(brush, backgroundPanel);
        ...
    }
    ...
}
```
cursor 변수를 통해 감지 작업을 수행하려고 하면 문제가 발생한다. cursor 변수에 들어있는 FocusWidget 객체는 복잡한 객체 생성 코드 내에 포함되어 있다. 이 객체를 생성하는 코드를 전부 클래스 외부로 옮길 수 있다면, 호출 코드는 객체를 생성하고 이를 이눗로서 전달할 수 있다.  
또는 인스턴스 변수 대체 기법이 있다. 객체를 생성한 후에 다른 인스턴스로 태체하기 위한 set메소드를 클래스에 추가하는 기법이다. 
```Cpp
class WatercolorPane {
public:
    WatercolorPane(Form *border, WashBrush *brush, Pattern *backdrop) {
        ...
        anteriorPanel = new Panel(border);
        anteriorPanel->setBorderColor(brush->getForeColor());
        backgroundPanel = new Panel(border, backdrop);
        cursor = new FocusWidget(brush, backgroundPanel);
        ...
    }
    ...

    void supersedeCursor(FocusWidget *newCursor) {
        delete cursor;
        cursor = newCursor;
    }
}


```
이제 대체 메소드가 완성되었으니 WatercolorPane 클래스 외부에서 FocusWidget 객체를 생성하고, 이를 객체 생성 후에 전달하는 것을 시도할 수 있다.  
그리고 인터페이스 추출 기법이나 구현체 추출 기법을 FocusWidget클래스에 적용하고, 가짜 객체를 생성해 전달함으로써 감지 작업을 수행할 수 있다.  
```Cpp
TEST(renderBorder, WatercolorPane) {
    ...
    TestingFocusWidget *widget = new TestingFocusWidget;
    WatercolorPane pane(form, border, backdrop);

    pane.supersedeCursor(widget);
    LONGS_EQUAL(0, pane.getComponentCount());
}
```
> 사실 이 방법 쫌 구리다고 한다..........

## 까다로운 전역 의존 관계
테스트 프레임워크에서 클래스 생성 및 사용을 어렵게 만드는 다양한 의존 관계들이 있다. 그중에서도 가장 까다로운 것이 전역 변수의 사용이다.  
다음은 정부 기관이 사용하는 건축 허가 관리 자바 애플리케이션의 클래스다.  
```Java
public class Facility {
    private Permit basePermit;

    public Facility(int facilityCode, String owner, PermitNotice notice) throws PermitViolation {
        Permit associatedPermit = PermitRepository.getInstance().findAssociatedPermit(notice);

        if (associatedPermit.isValid() && !notice.isValid()) {
            basePermit = associatedPermit;
        } else if (!notice.isValid()) {
            Permit permit = new Permit(notice);
            permit.validate();
            basePermit = permit;
        } else {
            throw new PermitViolation(permit);
        }
    }
    ...
}
```
테스트 하네스에서 Facility를 생성하고 싶으니 한번 시도해보자.
```Java
public void testCreate() {
    PermitNotice notice = new PermitNotice(0, "a");
    Facility facility = new Facility(Facility.RESIDENCE, "b", notice);
}
```
컴파일은 통과했지만 추가로 테스트 코드를 작성하면서 문제점을 깨닫게 된다. 생성자는 PermitRepository 클래스를 사용하기 떄문에 테스트를 제대로 수행하려면 일련의 허가(Permit 객체)들로 초기화해야 한다. 생성자 내에는 다음과 같은 말썽의 소지가 있는 문장이 있다.
```Java
Permit associatedPermit = PermitRepository.getInstance().findAssociatedPermit(notice);
```
> 이거 싱글톤임 ㅡ,.ㅡ  
  
싱글톤은 해당 정보가 어디에 어떻게 영향을 끼치는지 알기 매우 어렵기 때문에 바람직하지 않음.  
테스트 코드 내에서는 테스트 집합 내의 각개별 테스트 루틴을 하나의 작은 어플리케이션으로 봐야하기 떄문에 데이터들이 상호 영향을 주고 받으면 안된다.  
싱글톤의 제약을 풀어야한다.  
```Java
public class PermitRepository {
    private static PermitRepository instance = null;
    private PermitRepository() {}
    public static void setTestingInstance(PermitRepository newInstance) { // 이걸 추가하면 Testable해짐
        instance = newInstance
    }
    public static PermitRepository getInstance() {
        if (instance == null) {
            instance = new PermitRepository();
            return instance
        }
    }
    public Permit findAssociatedPermit(PermitNotice notice) {
        ...
    }
    ...
}
```
-> 테스트프레임워크 setup부분
```Java
public void setUp() {
    PermitRepository repository = new PermitRepository();
    ... 
    // 여기서 repository에 권한을 부여
    ...
    PermitRepository.setTestingInstance(repository);
}
```
아직 작동 안함.. 싱글톤 디자인 패턴을 사용할 때는 대체로 싱글톤 클래스의 생성자를 private으로 선언하는데, 싱글톤의 또 다른 인스턴스를 클래스 외부에서 생성하지 못하도록 막기 위해서다.  
여기서 설계 망함.. 인스턴스가 한개여야만 한다 vs 테스트를 위해 인스턴스를 여러번 생성한다.  
잠시 뒤를 돌아보아서 애당초 우리는 왜 시스템 내에 클래스의 인스턴스가 한 개만 생성되길 원하는가?
 1. 현실 세계를 모델링한 결과, 현실 세계에 한 개만 존재하기 때문
 2. 두 개가 존재한다면 심각한 문제가 발생하는 경우
 3. 두 개를 생성하면 자원 소모가 극심할 경우
  
이러한 이유로 싱글톤을 사용하지만 이것이 싱글톤을 사용하는 주요 이유는 아니다. 많은 사람들이 전역 변수를 갖기 위해 싱글톤을 생성하곤 한다. 변수를 필요한 장소로 여기저기로 전달하는 것을 힘들어 하기 떄문.  
만일 후자의 이유라면 싱글톤 없애야한다.  
  
어쨋든 여기선 public으로 생성자를 열어둠으로써 싱글톤 특성을 완화 하던가..  
아래와 같이 상속을 받자
```Java
public class TestingPermitRepository extends PermitRepository {
    private Map permits = new HashMap();
    public void addAssociatedPermit(PermitNotice notice, permit) {
        permit.put(notice, permit);
    }

    public Permit findAssociatedPermit(PermitNotice notice) {
        return (Permit)permits.get(notice);
    }
}
```
이렇게 씀으로써 PermitRepository의 생성자를 public이 아닌 protected까진 할 수 있다.  
  
인터페이스를 추출하면
```Java
public class PermitRepository implements IPermitRepository {
    private static PermitRepository instance = null;
    private PermitRepository() {}
    public static void setTestingInstance(PermitRepository newInstance) { // 이걸 추가하면 Testable해짐
        instance = newInstance
    }
    public static IPermitRepository getInstance() {
        if (instance == null) {
            instance = new PermitRepository();
            return instance
        }
    }
    public Permit findAssociatedPermit(PermitNotice notice) {
        ...
    }
    ...
}
```
요렇게도 가능


## 공포스러운 인클루드 의존관계
~~우선 C++에 대한 이야기..  
C++에서 한 클래스가 다른 클래스에 대해 알고 싶다면, 다른 파일에 들어있는 클래스 선언문을 텍스트 형태로 호출 측 파일에 인클루드 해야함. #include MyClass.h  
이 방식은 컴파일러가 이 선언문을 발견할 때마다 파싱을 다시 수행하고 빌드해야하기 때문에 오래걸린다.  
더욱 문제가 되는 것은 인클루드가 과도하게 사용되는 경향이 있음.~~
```Cpp
#ifndef SCHEDULER_H
#define SCHEDULER_H

#include "Meeting.h"
#include "MailDaemon.h"
...
#include "SchdulerDisplay.h"
#include "DayTime.h"
class Scheduler {
public:
    Scheduler(const string& owner);
    ~Scheduler();
    void addEvent(Event *event);
    bool hasEvents(Date date);
    bool performConsistencyCheck(string& message);
    ...
}
#endif
```
> 어떻게 테스트 루틴 내에서 Scheduler를 생성할 수 있을까?
  
~~가장 손 쉬운 방법은 동일 디렉토리에 SchedulerTests라는 파일 생성 후 빌드해봄 -> 전처리기 때문에~~
-> 너무 C++이라 그냥 건너뛰어도 될듯. 

## 양파껍질 매개변수 
모든 객체는 이후의 처리를 제대로 처리할 수 있도록 적절한 상태로 설정돼 있어야 한다. 이렇게 되려면 적절히 설정된 별도의 객체를 전달 받아야 할 떄가 많다. 그리고 이 객체의 설정을 위해 또 다른 객체가 필요하다. 따라서 테스트 대상 클래스의 생성자에 전달될 매개변수를 생성하기 위해 객체를 생성하고, 그 객체를 생성하기 위해 다른 객체를 생성하고 ~~~~~~  
객체 내에 또 다른 객체는 이른바 양파껍질 벗기기와 비슷하다.
```Java
public class SchedulingTaskPane extends SchedulerPane {
    public SchedulingTaskPane(SchedulingTask task) {
        ...
    }
}
```
이 클래스를 작성하려면 SchedulingTask 객체를 전달해야 한다. 그런데 SchedulingTask에 사용되는 생성자는 다음의 코드가 유일하다
```Java
public class SchedulingTask extends SerialTask {
    public SchedulingTask(Scheduler scheduler, MeetingResolver resolver) {
        ...
    }
}
```
> 또.. 필요하다.. ㅆ..?
  
이걸 어떻게 해야할까  
매개 변수 중에서 테스트에 불필요한것은 Null 전달 기법을 사용하자.  
몇 개의 기본적인 동작만 필요하다면, 직접 의존 관계를 갖는것에 인터페이스 추출이나 구현체 추출 기법을 사용해 인터페이스를 통한 가짜 객체를 생성할 수 있을것이다.  
![KakaoTalk_Photo_2023-01-09-23-24-20](https://user-images.githubusercontent.com/60125719/211330534-33d6b8f7-2116-421d-b040-6a1f9254f7fe.jpeg)

```Cpp
class SerialTask {
    public:
    virtual void run();
    ...
};

class ISchedulingTask {
public: 
    virtual void run() = 0;
    ...
};

class SchedulingTask: public SerialTask, public ISchedulingTask {
    public virtual void run() { SerialTask::run(); }
}

```
> 대충 C++로 되어있는 예제.. Cpp은 인터페이스 라는게 없어서 ㅡㅡ;; 이렇게 해야한다고 한다 ㅡ,.ㅡ;; 뭐 우리랑은 상관 없는 얘기일듯

## 별명을 갖는 매개변수
생성자의 매개변수가 문제가 되는 경우에는 대체로 인터페이스 추출이나 구현체 추출기법을 사용해 문제를 우회할 수 있다. 그러나 가끔 안될때도 있다.
```Java
public class IndustrialFacility extends Facility {
    Permit basePermit;

    public IndustrialFacility(int facilityCode, String owner, OriginationPermit permit) throws PermitViolation {
        Permit associatedPermit = PermitRepository.getInstance().findAssociatedFromOrigination(permit);

        if (associatedPermit.isValid() && !permit.isValid()) {
            basePermit = associatedPermit
        } else if (!permit.isValid()) {
            permit.validate();
            basePermit = permit;
        } else {
            throw new PermitViolation(permit);
        }
    }
}
```
테스트 하네스에서 이 클래스를 생성하는 데 몇 가지 문제가 있다. 
1. PermitRepository 싱글톤에 접근하고 있다. -> 앞에서 배운 '까다로운 전역 의존 관계'절에서 설명한 기법으로 처리할 수 있다.
2. 생성자에게 전달해야 하는 OriginationPermit 객체를 생성하기가 어렵다. (OriginationPermit 이 의존관계가 복잡함) -> 인터페이스를 추출해 의존관계를 제거하면 되겠구나?! -> 확인해보자
![KakaoTalk_Photo_2023-01-09-23-37-42](https://user-images.githubusercontent.com/60125719/211333532-a01280a9-3a2a-4d89-8c83-a8f038bbee4c.jpeg)
IndustrialFacility 생성자는 OriginationPermit을 받아서 PermitRepository로부터 관련 Permit을 얻기위해 PermitRepository로부터의 메소드를 사용한다.  
관련 Permit을 발견하면 이를 basePermit에 저장하고, 못찾으면 OriginationPermit을 basePermit필드에 저장한다.  
인터페이스들로만 이뤄진 계층 구조를 생성한 후 Permit필드를 IPermit필드로 변환하자.  
![KakaoTalk_Photo_2023-01-09-23-41-53](https://user-images.githubusercontent.com/60125719/211334397-9eaa2b1f-3360-486f-8431-74ec98c13fc9.jpeg)
> 그런데 이거 너무 어려움..  
  
인터페이스는 의존관계 제거에는 효과적이지만 이방법뿐이 없는것은 아니다.  
해당 클래스와의 연결을 그냥 제거하면 될 떄도 있다. 
```Java
public class OriginationPermit extends FacilityPermit {
    ...
    public void validate() {
        // 데이터베이스 연결
        ...
        // 정보 검증 질의
        ...
        // 확인 플래그 설정
        ... 
        // 데이터베이스 연결 해제
    }
}
```
> validate에서 디비에 접근하고 있다.  
  
테스트 중에 이 메소드를 실행시키고 싶지 않음.  
이 상황에선 서브클래스화 메소드 재정의 기법을 써볼만하다.  
FakeOriginationPermit이라는 클래스를 만들어서 검증 플래그를 쉽게 변경할 수 있는 메소드를 제공한 후, 서브클래스에서 재정의된 validate 메소드를 IndustrialFacility 클래스를 테스트하는 동안에 사용함으로써 검증 플래그를 변경할 수 있다.
```Java
public void testHasPermits() {
    class AlwaysValidPermit extends FakeOriginationPermit {
        public void validate() {
            //확인플래그 설정
            becomeValid();
        }
    };

    Facility facility = new IndustrialFacility(Facility.HT_1, "b", AlwaysValidPermit());
    assertTrue(facility.hasPermit());
}
```
































