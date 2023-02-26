# 18. 테스트 코드가 방해를 한다.
## 클래스 명명 규칙
일반적으로, 각 클래스마다 적어도 한 개의 단위테스트 클래스를 작성한다. 따라서 테스트 대상 클래스의 이름을 바탕으로 단위 테스트 클래스 이름을 만드는 것은 합리적인 판단이다.  
가장 보편적인 규칙은 클래스 이름의 접두어나 접미어로 Test를 붙이는것이다. 예를들어 DBEngine/TestDBEngine/DBEngineTest  
가짜 클래스의 이름은 접두어로 Fake를 붙힘(글 쓴 사람의 경우)  
테스트용 서브 클래스가 필요할 수도 있다. 이럴떈 접두어로 붙힘  
ex) 간단한 회계용 소프트웨어의 클래스 목록
 - CheckingAccount 
 - CheckingAccountTest 
 - FakeAccountOwner
 - SavingAccount
 - SavingAccountTest
 - TestingCheckingAccount
 - TestingSavingAccount

-> 여튼 어떤 규칙이 되었든 일관성있게 만들자.

## 테스트 코드의 배치
테스트 코드가 배포바이너리에 포함됨? 잘 고려해서 넣어야한다.(자바는 이러나)
```
source
    come 
        orderprocessing
            dailyorders
test
    come
        orderprocessing
            dailyorders
```
> 이런식으로 디렉토리 관리를 해보자.  
  





























