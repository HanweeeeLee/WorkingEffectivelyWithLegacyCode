# 03. 감지와 분리
이상적인 환경이라면 변경 작업을 하기 전 특별히 할 일이 없다.  
테스트를 하고 싶은 대상을 테스트 코드 내에서 객체를 생성해 곧바로 작업을 하면 된다.  
하지만 불행하게도 클래스들의 의존 관계들 때문에 이것은 거의 불가능하다.  
의존관계를 제거해야 하는 이유는 무엇일까? 크게 두가지로 볼수 있다.
1. 감지 : 코드 내에서 계산된 값에 접근하여 변경이나 값을 확인할 수 없을 때
2. 분리 : 코드를 테스트 코드 내에서 실행할 수 없을 때 코드 분리를 위해

다음의 예제를 보자 네트워크 관리 애플리케이션에 NetworkBridge라는 클래스가 있다.
```Java
public class NetworkBridge {
    public NetworkBridge(EndPoint[] endpoints){
        ...
    }

    public void formRouting(String sourceId, String destId){
        ...
    }
...
}
```
위 코드에 대한 간략한 설명은
1. NetworkBridge 클래스는 EndPoint 배열을 입력받고, 하드웨어에 전달하여 트래픽을 관리한다.
2. 사용자는 해당 클래스의 메소드를 사용하여 종점 간 트래픽 경로를 제어할 수 있다.
3. NetworkBridge는 이러한 작업들을 진행하기 위해 전달받은 EndPoint 설정을 변경하고 Endpoint 클래스의 인스턴스들은 소켓을 열고 네트워크를 통해 특정 장치와 통신할수 있다.

테스트 관점에서 몇가지 문제점들이 눈에 보이는데,
1. 이 클래스의 객체는 실제 하드웨어를 자주 호출할텐데, 테스트를 하기 위해 실제 하드웨어가 필요한가?
2. NetworkBridge 내부에서 통신을 위해 EndPoint 객체에 무슨 작업을 했을까? → 블랙박스이기에 내부 작업을 들여다 볼수 없다.

이것이 문제가 되지 않을수도 있다.  
네트워크상의 패킷을 읽을수 있는 코드를 작성할 수 있고, 실제 하드웨어를 구입할수도 있다.  
또는 배선작업을 통해 네트워크 종점들의 로컬 클러스터를 구성해 테스트에 사용할 수도 있다.  
문제는 이러한 방법들이 해결책이 될수는 있지만, 엄청난 노력과 비용이 발생하게 된다는 것이다.  

이 예제는 감지와 분리 문제를 모두 보여주는데, 클래스의 메소드가 호출될 때의 영향을 감지할 수도 없고, 애플리케이션의 나머지 부분과 분리해서 실행할 수도 없기 때문이다.  
감지와 분리 중 어느쪽이 더 어려운 문제인지에 대한 명확한 답은 없다.  
다만 분명한 것은 분리에 사용되는 기법은 매우 다양하고, 감지의 경우에는 거의 언제나 다음 기법을 사용한다.

## 협업 클래스 위장하기
레거시 코드를 다룰 때의 가장 큰 문제 중 하나가 의존관계이다.  
특정 코드만 독립적으로 실행해 어떻게 동작하는지 테스트하려면, 대체로 다른 코드에 대한 의존관계를 제거할 필요가 있지만, 이것은 간단한 일이 아니다.  
보통 그 다른 코드가 우리가 수행하는 작업의 영향을 감지할수 있는 유일한 위치일때가 많기 때문이다.  
따라서 그 다른 코드를 별도의 코드로 대체할수 있다면, 변경 대상을 테스트하는 루틴을 작성할수 있을 것이고, 객체 지향 프로그래밍에서는 이 별도의 코드를 가리켜서 가짜객체 혹은 위장객체(fake object) 라고 부른다.

### 가짜 객체
가짜 객체란 어떤 클래스를 테스트할 때 그 클래스의 협업 클래스를 모방하는 객체를 말한다.
다음은 POS시스템에 Sale이란 클래스가 있고, 이 클래스의 scan()메소드는 고객이 구매중인 품목의 바코드를 읽어 해당품목의 이름과 가격을 디스플레이 화면에 출력하는 예제이다.  
이때 화면에 올바르게 표시되는지 여부를 어떻게 테스트 할 수 있을까?  
코드 내부의 어느곳에서 화면 갱신을 수행하는지 정확히 구분할 수 있다면 그림 3.2처럼 설계를 변경할 수 있다.
![KakaoTalk_20221128_175821200_01](https://user-images.githubusercontent.com/50142323/204240226-fa0a6a2c-8099-4806-b818-8c54579492c5.jpg)

ArtR56Display라는 새로운 클래스가 추가된 것을 볼 수 있다.  
이 클래스는 현재 사용중인 디스플레이 장치와 통신하는 데 필요한 모든 코드를 포함하고 있고 화면에 표시될 텍스트를 이 클래스에 전달하기만 하면 된다.  
Sale 클래스 내부에 들어있던 화면 표시와 관련된 코드를 ArtR56Display클래스로 옮겨도 시스템의 동작은 이전과 달라질 것이 없기 때문이다.  
이러한 코드 이동은 그림 3.3과 같은 설계가 가능하다는 장점을 가지게 된다.
![KakaoTalk_20221128_175821200](https://user-images.githubusercontent.com/50142323/204240894-33470c74-04d0-451a-be0d-b098fe358f28.jpg)

이제 Sale 클래스는 ArtR56Display 클래스 뿐 아니라 FakeDisplay 클래스와도 통신 할 수 있게 되고, 이 FakeDisplay 클래스는 Sale클래스가 무슨 동작을 하는지 확인하기 위한 테스트 루틴을 작성할 수 있다.  
Sale 객체에 인수로서 전달되는 디스플레이 객체는 Display 인터페이스를 구현하는 클래스이기만 하면 어떤 클래스의 객체도 가능하기 때문이다.

```Java
public class Display {
    void showLine(String line);
}
```

ArtR56Display와 FakeDisplay 모두 Display 인터페이스를 구현한다.  
Sale객체는 생성자를 통해 디스플레이를 전달받고 내부적으로 이를 유지할 수 있다.

```Java
public class Sale {
    private Dispaly display;
    
    public Sale(Display display){
        this.display = display;
    }
    
    public void scan(String barcode){
        ...
        String itemLine = item.name()
            + " " + item.price().asDisplayText();
        display.showLine(itemLine);
        ...
    }
}
```

scan 메소드는 display 변수의 showLine 메소드를 호출하고, 어떤 동작이 수행될 지는 Sale클래스에 전달한 디스플레이가 무엇인지에 따라 다르다.  
다음 코드는 이러한 확인에 사용될수 있는 테스트 코드 예제 이다.
```Java
inport junit.framework.*
public class SaleTest extends TestCase {
    public void testDisplayAnItem(){
        FakeDisplay display = new FakeDisplay();
        Sale sale new Sale(display);
        
        sale.scan("1");
        assertEquals("Milk $3.99", display.getLastLine())
    }
}
```

FakeDisplay 클래스는 다소 독특한 면이 있다. 자세히 살펴보자.
```Java
public class FakeDisplay implements Display {
    private string lastLine = "";
    
    public void showLine(String line) {
        lastLine = line;
    }
    
    public String getLastLine() {
        return lastLine;
    }
}
```

showLine 메소드는 한줄의 텍스트를 받아서 lastLine변수에 대입한다. 그리고 getLastLine메소드는 호출될 때마다 이 텍스트를 반환한다.  
앞서 작성한 테스트 루틴을 사용하면 Sale클래스가 사용될 때 텍스트가 제대로 디스플레이로 전달됬는지 확인할 수 있다.

### 가짜 객체의 양면성
객체는양면성, 즉 두가지 측면을 가진다는 점을 이해하기 쉽지 않다.  
그림 3.4에서 FakeDisplay 클래스를 다시 한 번 살펴보자.
![KakaoTalk_20221128_185642432](https://user-images.githubusercontent.com/50142323/204248733-3600402a-571d-4f4b-ae82-9bf233fe1664.jpg)

FakeDisplay 클래스는 Display 인터페이스를 구현하기 떄문에 showLine 메소드가 필요하다.  
이 메소드는 Display 인터페이스의 유일한 메소드이자 Sale 클래스가 바라볼 수 있는 유일한 메소드이고, 또 하나의 메소드인 getLastLine은 테스트용 이다.  
그래서 변수를 Display가 아니라 FakeDisplay 타입으로 선언한 것 이다.
```Java
inport junit.framework.*
public class SaleTest extends TestCase {
    public void testDisplayAnItem(){
        **FakeDisplay** display = new FakeDisplay();
        Sale sale new Sale(display);
        
        sale.scan("1");
        assertEquals("Milk $3.99", **display.getLastLine()**)
    }
}
```
