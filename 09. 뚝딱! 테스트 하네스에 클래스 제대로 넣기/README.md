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






































