# 19. 내 프로젝트는 객체 지향이 아니다. 어떻게 안전하게 변경할 수 있을까?
절차형 언어는 레거시 환경에서 특히 까다롭다. 코드를 변경하기 전에 테스트 루틴을 만드는 것이 중요한데, 절차형 언어는 단위 테스트를 도입하기 위해 할 수 있는 일이 제한적이기 때문이다.  
절차형 코드는 의존관계를 제거하기가 매우 어려운데, 최상의 전략은 변경을 수행하기 전에 먼저 대규모 코드 묶음을 테스트 루틴에 넣고 이를 이용해 개발 과정에서 피드백을 얻는 것이다.  

## 간단한 경우
다음은 리눅스에서 동작하는 C함수의 예이다

```C
void set_writetime(struct buffer_head * buf, int flag)
{
  int newtime;
  if(buffer_dirty(buf)) {
    /* Move Bbuffer to dirty list if jiffies is clear */
    newtime = jiffies + (flag? bdf_prm.b_un.age_super : bdf_prm.b_un.age_buffer);
    
    if(!buf->b_flushtime || buf->b_flushtime > newtime)
      buf->b_fluishtime = newtime;    
  } else {
    buf->b_flushtime = 0;
  }
}
```
이 함수를 테스트하기 위해 필요한 과정은
 - jiffies 변수 값을 설정
 - buffer_head를 생성해 함수에 전달
 - 호출 후 buffer_head의 값을 검사

하지만 이렇게 운이 좋은 경우는 그다지 없다. 함수내에서 다른 함수를 호출하고 그 호출된 함수는 또다른 함수를 호출하는 식으로 계속 이어지는 경우가 대다수이다.  

## 어려운 경우
```C
#include "ksrlib.h"
int scan_packets(struct rnode_packet *packet, int flag)
{
  struct rnode_packet *current = packet;
  int scan_result, err = 0;
  
  while(current) {
    scan_result = loc_scan(current->body, flag);
    if(scan_result & INVALID_PORT) {
      ksr_notify(scan_result, current);
    }
    ...
    current = current->next;
  }
  
  return err;
}
```
이 경우 ksr_notify라는 함수를 호출하는데 이 함수는 부작용을 포함하고 있다. 서드파티 시스템에 알림을 보내는데, 테스트 중에는 알림을 보내지 않고 싶기 때문이다.  
이 문제를 처리하는 한가지 방법으로 연결 봉합을 사용해 라이브러리 함수의 영향을 받지 않기 위해 가짜 함수를 포함하는 라이브러리를 만드는 것이다.  
이번의 경우 가짜함수는 다음과 같다.  
```C
void ksr_notify(int scan_code, struct rnode_packet *packet)
{  
}
```
이 함수를 라이브럴리 안에서 빌드하고 링크 시킬수 있고, scan_packets 함수는 알림을 보내지 않는 다는점만 제외하고 똑같이 동작할 것이다.  
이러한 전략을 사용해야 할까? 경우에 따라 다른데 ksr 라이브러리에 많은 함수들이 있고 이 함수들의 호출이 시스템의 주요 처리가 아니라면 이 전략을 사용할 만하다.
반면 함수를 통해 어떤것을 감지하길 원하거나 함수가 반환하는 값을 변경하고 싶다면 이는 좋은 전략이 아니다.  
연결봉합을 사용하면 링크하는 시점에서 라이브러리를 교체하기 때문에, 빌드된 실행파일마다 함수정의를 한개만 가질 수 있다.  
따라서 가짜 함수가 어떤테스트에서는 이렇게 다른 테스트에서는 저렇게 동작하길 원한다면, 함수 본문에 특정 실행방법을 조건문으로 설정해야만 한다.  
불행히도 대부분의 절차형 언어에서는 다른 선택지가 없다. C언어라면 대안이 한가지 있는데 전처리기를 제공하므로 이것을 사용해 테스트 루틴을 쉽게 작성할 수 있다.

```C
#include "ksrlib.h"
#ifdef TESTING
#define ksr_notify(code,packet)
#endif
int scan_packets(struct rnode_packet *packet, int flag)
{
  struct rnode_packet *current = packet;
  int scan_result, err = 0;
  
  while(current) {
    scan_result = loc_scan(current->body, flag);
    if(scan_result & INVALID_PORT) {
      ksr_notify(scan_result, current);
    }
    ...
    current = current->next;
  }
  
  return err;
}

#ifdef TESTING
#include <assert.h>
int main() {
  struct rnode_packet packet;
  packet.body = ...
  ...
  int err = scan_packets(&packet, DUP_SCAN);
  assert(err & INVALID_PORT);
  ...
  return 0;
}
#endif
```
TESTING이라는 매크로 정의를 사용해 테스트할때 ksr_notify 호출을 무효화한다. 또한 짧은 테스트 루틴도 포함하고 있다.  
이처럼 테스트코드와 배포코드를 혼합해 한개의 파일에 넣으면 코드를 이해하기 어려울수 있기 때문에, 인클루드 기능을 사용해 테스트 코드와 배포 코드를 별도의 파일들에 두는 방법을 대안으로 사용할 수 있다.  

```C
#include "ksrlib.h"
#include "scannertestdefs.h"
int scan_packets(struct rnode_packet *packet, int flag)
{
  struct rnode_packet *current = packet;
  int scan_result, err = 0;
  
  while(current) {
    scan_result = loc_scan(current->body, flag);
    if(scan_result & INVALID_PORT) {
      ksr_notify(scan_result, current);
    }
    ...
    current = current->next;
  }
  
  return err;
}

#include "testscanner.tst"
```
테스트 대상 함수에 전방 선언을 사용하면 하단에 있는 인클루드 파일의 모든 내용을 상단의 인클루드 파일로 이동시킬 수 있다.  
테스트를 실행하기 위해서는 TESTING을 정의하고 이 파일만 빌드 하면 된다.  
이때 작성된 테스트용 함수들은 다른 파일의 main 함수에서 호출할 수 있다.  
C언어에서는 이러한 식으로 처리가 가능하지만 다른 절차형 언어에서는 일반적으로 연결 봉합을 사용해 의존관계를 제거하면서 대규모 코드에 대한 테스트 루틴을 작성해야 할 것이다.  


## 새로운 동작의 추가
절차형 언어의 경우 기존 함수에 코드를 추가하는 것 보다 새로운 함수를 도입하는 편이 낫다. 최소한 새로 작성한 함수를 위한 테스트 루틴을 작성할 수 있기 때문이다.  
절차형 언어의 코드에서 의존 관계의 함정에 빠지지 않으려면 어떻게 해야 할까?  
한가지 방법으로는 tdd를 사용하는 것이다. 작성하려는 코드에 대해 테스트 루틴을 분명히 함으로서 설계를 바람직한 방향으로 이끌 수 있다.  
TDD에서는 함수를 작성하는 데 집중하고 이 함수를 실행하면서 애플리케이션의 나머지 부분과 통합해 나간다.
```C
void send_command(int id, char *name, char *command_string)
{
  char *message, *header;
  if(id == KEY_TRUM) {
    message = ralloc(sizeof(int) + HEADER_LEN + ...
    ...
  } else {
    ...
  }
  sprintf(message, "%s%s%s", header, command_string, footer);
  mart_key_send(message);
  free(message);
}
```
이 함수는 mart_key_send 함수를 통해 iD, 이름, 명령 을 다른 시스템으로 보낸다.  
mart_key_send 호출 이전의 모든 처리가 별도의 함수에 작성돼 있다면, 이 함수의 테스트 코드를 작성할수 있다.
```C
char *command = form_command(1, "Mike Ratledge", "56:78:cusp-:78");
assert(!strcmp("<-rsp-Mike Ratledge><56:78:cusp-:78><-rspr>", command));
```
이어서 명령 문자열을 반환하는 form_command 함수를 작성한다
```C
char *form_command(int id, char *name, char *command_string)
{
  char *message, *header;
  if(id == KEY_TRUM) {
    message = ralloc(sizeof(int) + HEADER_LEN + ...
    ...
  } else {
  ...
  }
  sprintf(message, "%s%s%s", header, command_string, footer);
  return message;
}
```
이 함수를 이용하면 send_command 함수는 다음과 같이 단순화 된다.
```C
void send_command(int id, char *name, char *command_string)
{
  char *command = form_command(id, name, command_string);
  mart_key_send(command);
  free(message);
}
```
이처럼 코드의 구성을 바꾸는 작업이 의존관계를 제거하여 앞으로 나아가기 위해 필요할 때가 많다.  
이렇게 되면 결국은 send_command 같은 래퍼 함수가 만들어진다. 래퍼 함수는 처리로직과 의존관계가 결합돼 있다.  
이 기법은 만능은 아니지만 의존관계가 광범위하게 퍼지지 않은 상황에서는 쓸만하다.  
또한 다수의 외부호출을 수행하는 함수를 작성해야 할 경우가 있는데, 이런 함수는 계산을 그다지 수행하지 않지만 함수 호출 순서는 매우 중요하다.  
다음은 대출 이자를 계산하는 함수이다.
```C
void calculate_loan_interest(struct temper_loan *loan, int calc_type)
{
  ...
  db_retrueve(loan->id);
  ...
  db_retrieve(loan->lender_id);
  ...
  db_update(loan->id, loan->record);
  ...
  loan->interest = ...
}
```
다수의 절차형 언어에서 가장 좋은 방법은 테스트 루틴 작성을 건너뛰고, 최선을 다해 함수를 작성하는 것이다.  
하지만 C언어는 다른 선택지가 있는데, C언어는 함수 포인터를 지원하므로, 함수 포인터를 봉합부로서 사용할 수 있다.  
먼저 함수 포인터를 포함하는 구조체를 생성한다.
```C
struct database
{
  void (*retrieve)(struct record_id id);
  void (*update)(struct record_id id, struct record_set *record);
  ...
}
```
이 함수 포인터들은 데이터베이스 접근 함수의 주소로 초기화된다. 이구조체는 DB에 접근하려는 다양한 함수들에 전달할 수 있다.  
배포 코드에서는 실제 DB 접근함수를 가리키고, 테스트코드에서는 가짜 함수를 가리키도록 설정한다.  
초창기 컴파일러를 사용중이라면 고전적인 함수 포인터 구문을 사용해야 하지만 최근의 컴파일러를 사용중이라면 객체 지향 스타일의 자연스러운 구문으로 호출 할 수 있다.  
```C
extern struct database db;
db.update(load->, loan->record);
```
이러한 기법은 C에서만 사용할 수 있는 것은 아니고 함수 포인터와 위임 기능을 지원하는 대부분의 언어에서 이 기법을 사용할 수 있다.

## 객체 지향의 장점 이용
객체 지향 언어의 경우, 객체 봉합 기법을 사용할 수 있다. 객체 봉합은 다음과 같은 장점들이 있다.  
 - 코드 내에서 발견하기 쉽다.
 - 코드를 작고 이해하기 쉬운 부분들로 분해할 수 있다.
 - 유연성이 높다. 테스트를 위해 도입한 봉합부는 소프트웨어를 확장할 때도 유용하게 쓰일 수 있다.

모든 소프트웨어가 객체 지향으로 간단히 마이그레이션 될 수는 없지만, 매우 간단히 되는 경우도 있다.   
많은 절차형 언어들이 객체 지향 언어로 진화해왔기 때문이다.  
사용 중인 언어가 객체 지향으로 옮겨갈 수 있는 방법을 제공하고 있다면 우리에게는 더 많은 선택지가 주어진다.  
대부분의 경우, 가장 먼저 할일은 전역참조 캡슐화 기법을 사용하여 변경대상 코드를 테스트 코드 안에 넣는 것이다.  
이러한 방법은 매우 안정저깅며 상당 부분을 기계적으로 수행할 수 있다.  
객체 지향 설계의 좋은 사례는 아니지만 의존관계를 제거하기에 충분한 역할을 하므로 작업을 진행함에 따라 테스트 범위가 계속 증가할 것이다.

## 모든 것이 객체 지향적이다
절차형 언어 프로그래머 중에는 객체지향은 불필요하다거나 복잡성에 비하면 그다지 유용하지 않다고 주장한다.  
하지만 절차형 프로그램도 엄밀히 말하자면 객체지향적이다. 단지 단 하나의 객체만 갖고 있는 것일 뿐이다.  
사용 중인 언어가 객체 지향에 대응할 경우 의존관계 추출 이외에 어떤 일을 할 수 있을까?  
한가지를 꼽자면 바람직한 객체 지향 설계를 목표로 조금씩 개선하는 것이다. 일반적으로는 서로 관련된 함수들을 한개의 클래스로 모은 후, 다수의 메소드를 추출함으로서 복잡하게 얽힌 책임들을 분할하는 것을 의미한다.  
절차형 코드는 객체 지향 코드만큼 많은 선택지를 제공하지는 않는다. 하지만 절차형 레거시 코드를 개선할 수는 있다.  
현재 사용중인 절차형 언어의 후속 언어로서 객체 지향 언어가 있다면 그 객체지향 언어로 마이그레이션할 것을 권장한다.

