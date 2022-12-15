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
지금까지 설명한 것은 발아 메소드(sprout method)의 예다. 발아 메소드의 작성 순서는 다음과 같다.  
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














































