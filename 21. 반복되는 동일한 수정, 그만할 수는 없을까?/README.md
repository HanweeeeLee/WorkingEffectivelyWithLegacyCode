# 반복되는 동일한 수정, 그만할 수는 없을까?
코드를 변경한 후 "이제 됐군."이라고 생각했는데, 사실은 시스템 내의 여기저기에 유사한 코드가 분산돼 있으므로 동일한 변경을 반복해야 한다는 것을 알게 된다.  
리팩토링을 알고있다면 그나마 다행. 여기서 핵심적인 질문은 "과연 리팩토링을 할 가치가 있는가?", "중복코드를 열정적으로 제거하면 무슨 이득이 있는가?"  
```Java
import java.io.OutputStream;

public class AddEmplyoyeeCmd {
    String name;
    String address;
    String city;
    String state;
    String yearlySalary;
    private static final byte[] header = {(byte)0xde, (byte)0xad};
    private static final byte[] commandChar = {0x02};
    private static final byte[] footer = {(byte)0xb2, (byte)0xef};
    private static final int SIZE_LENGTH = 1;
    private static final int CMD_BYTE_LENGTH = 1;

    private int getSize() {
        return header.length + 
            SIZE_LENGTH + 
            CMD_BYTE_LENGTH +
            footer.length +
            name.getBytes().length + 1 +
            address.getBytes().length + 1 +
            city.getBytes().length + 1 +
            state.getBytes().length + 1 +
            yearlySalary.getBytes().length + 1;
    }

    public AddEmplyoyeeCmd(String name, String address, String city, String state, int yearlySalary) {
        this.name = name;
        this.address = address;
        this.city = city;
        this.state = state;
        this.yearlySalary = Integer.toString(yearlySalary);
    }

    public void write(OutputStream outputStream) throws Exception {
        outputStream.write(header);
        outputStream.write(getSize());
        outputStream.write(commandChar);
        outputStream.write(name.getBytes());
        outputStream.write(0x00);
        outputStream.write(address.getBytes());
        outputStream.write(0x00);
        outputStream.write(city.getBytes());
        outputStream.write(0x00);
        outputStream.write(state.getBytes());
        outputStream.write(0x00);
        outputStream.write(yearlySalary.getBytes());
        outputStream.write(0x00);
        outputStream.write(footer);
    }

}


import java.io.OutputStream;

public class LoginCommand {

    private String userName;
    private String passwd;
    private static final byte[] header = {(byte)0xde, (byte)0xad};
    private static final byte[] commandChar = {0x01};
    private static final byte[] footer = {(byte)0xb2, (byte)0xef};
    private static final int SIZE_LENGTH = 1;
    private static final int CMD_BYTE_LENGTH = 1;

    public LoginCommand(String userName, String passwd) {
        this.userName = userName;
        this.passwd = passwd;
    }

    private int getSize() {
        return header.length + SIZE_LENGTH + CMD_BYTE_LENGTH + 
            footer.length + userName.getBytes().length + 1 + 
            passwd.getBytes().length + 1;
    }

    public void write(OutputStream outputStream) throws Exception {
        outputStream.write(header);
        outputStream.write(getSize());
        outputStream.write(commandChar);
        outputStream.write(userName.getBytes());
        outputStream.write(0x00);
        outputStream.write(passwd.getBytes());
        outputStream.write(0x00);
        outputStream.write(footer);
    }

}
```
> 자바 기반의 소규모 네트워크 시스템을 통해 서버에 명령어를 보낸다.

![KakaoTalk_Photo_2023-03-25-20-18-33](https://user-images.githubusercontent.com/60125719/227714358-a2f0b389-18bd-4d77-a71f-268e239677fc.jpeg)
> UML 클래스 다이어그램으로 이 두개의 클래스를 표현.
  
중복코드를 제거하면 어떻게 될까? 좋아질까?  
리팩토링 해보자. 테스트코드가 이미 작성되었다고 가정  

## 첫 번째 단계
전체적인 모습을 파악하고, 어떤 클래스로 할지, 추출된 중복코드를 어떻게 할지 검토.  
소규모 중복 부분을 제거하는 것만으로도 충분히 효과적
  
ex)
```Java
// LoginCommand Class의 write method 
outputStream.write(userName.getBytes());
outputStream.write(0x00);
outputStream.write(passwd.getBytes());
outputStream.write(0x00);
```
문자열을 생성할 떄 널 문자(0x00)를 맨 끝에 추가해야하는데 중복코드로 보고 이렇게 할 수 있음
```Java
void writeField(OutputStream outputStream, String field) {
    outputStream.write(field.getBytes());
    outputStream.write(0x00);
}
```
이렇게 하면 다음과 같이 개선할 수 있다.
```Java
// LoginCommand Class의 write method 
public void write(OutputStream outputStream) throws Exception {
    outputStream.write(header);
    outputStream.write(getSize());
    outputStream.write(commandChar);

    // outputStream.write(userName.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, userName);
    // outputStream.write(passwd.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, passwd);

    outputStream.write(footer);
}
```
여전히 AddEmplyoyeeCmd 클래스 내부는 같은 문제를 앉고있다.  
두개의 클래스는 모두 명령어 클래스이므로, Command라는 슈퍼클래스를 도입하자.
![KakaoTalk_Photo_2023-03-25-20-25-50](https://user-images.githubusercontent.com/60125719/227714663-75578e40-34c3-46c3-a4d4-a2af027c32a2.jpeg)
  
이제 AddEmplyoyeeCmd를 개선해보자.
```Java
public void write(OutputStream outputStream) throws Exception {
    // AddEmplyoyeeCmd Class
    outputStream.write(header);
    outputStream.write(getSize());
    outputStream.write(commandChar);

    // outputStream.write(name.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, name);
    // outputStream.write(address.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, address);
    // outputStream.write(city.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, city);
    // outputStream.write(state.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, state);
    // outputStream.write(yearlySalary.getBytes());
    // outputStream.write(0x00);
    writeField(outputStream, yearlySalary);

    outputStream.write(footer);
}
```

아직 끝나지 않았다. AddEmplyoyeeCmd, LoginCommand의 write 메소드는 굴 다 헤더와 크기를 기록하고, 명령어 문자를 기록한 후 일련의 필드를 기록하고, 마지막으로 푸터를 기록한다. 따라서 둘 간의 다른 부분, 즉 필드를 기록하는 코드 부분을 추출해보자.

```Java
// LoginCommand Class
private void writeBody(OutputStream outputStream) throws Exception {
    writeField(outputStream, userName);
    writeField(outputStream, passwd);
}

public void write(OutputStream outputStream) throws Exception {
    outputStream.write(header);
    outputStream.write(getSize());
    outputStream.write(commandChar);

    // writeField(outputStream, userName);
    // writeField(outputStream, passwd);
    writeBody(outputStream);

    outputStream.write(footer);
}
```
```Java
// AddEmplyoyeeCmd Class
private void writeBody(OutputStream outputStream) throws Exception {
    writeField(outputStream, name);
    writeField(outputStream, address);
    writeField(outputStream, city);
    writeField(outputStream, state);
    writeField(outputStream, yearlySalary);
}

public void write(OutputStream outputStream) throws Exception {
    outputStream.write(header);
    outputStream.write(getSize());
    outputStream.write(commandChar);

    // writeField(outputStream, name);
    // writeField(outputStream, address);
    // writeField(outputStream, city);
    // writeField(outputStream, state);
    // writeField(outputStream, yearlySalary);
    writeBody(outputStream);

    outputStream.write(footer);
}
```

이제 공통으로 wirteBody 사용가능? ㄴㄴ아직 불가능 -> Footer, header, commandChar가 다르다.  
클래스의 header, footer, SIZE_LENGTH, CMD_BYTE_LENGTH 변수들은 같은 값을 갖고 있으므로 Command 클래스로 올릴 수 있다. 컴파일 및 테스트가 잘 되도록 일시적으로 protected로 선언해야 한다.  
```Java
public class Command {
    protected static final byte[] header = {(byte)0xde, (byte)0xad};
    protected static final byte[] footer = {(byte)0xb2, (byte)0xef};
    protected static final int SIZE_LENGTH = 1;
    protected static final int CMD_BYTE_LENGTH = 1;

    ...
}

```
이제 서브클래스들에는 commandChar변수가 남아있다. 서로 다른 값을 갖고 있는데, Command클래스에 추상 get 메소드를 도입하는것으로 해결가능
```Java
public class Command {
    protected static final byte[] header = {(byte)0xde, (byte)0xad};
    protected static final byte[] footer = {(byte)0xb2, (byte)0xef};
    protected static final int SIZE_LENGTH = 1;
    protected static final int CMD_BYTE_LENGTH = 1;

    protected abstract char [] getCommandChar();

    ...
}

public class AddEmplyoyeeCmd extends Command {
    // private static final byte[] commandChar = {0x02};
    protected char [] getCommandChar() {
        return new Char [] {0x02};
    }
    ...
}

public class LoginCommand extends Command {
    // private static final byte[] commandChar = {0x02};
    protected char [] getCommandChar() {
        return new Char [] {0x01};
    }
    ...
}
```

이제 write 메소드를 슈퍼클래스로 끌어올려도 된다.
```Java
public class Command {
    protected static final byte[] header = {(byte)0xde, (byte)0xad};
    protected static final byte[] footer = {(byte)0xb2, (byte)0xef};
    protected static final int SIZE_LENGTH = 1;
    protected static final int CMD_BYTE_LENGTH = 1;

    protected abstract char [] getCommandChar();
    protected abstract void writeBody(OutputStream outputStream);
    protected void writeField(OutputStream outputStream, String field) {
        outputStream.write(field.getBytes());
        outputStream.write(0x00);
    }
    public void write(OutputStream outputStream) throws Exception {
        outputStream.write(header);
        outputStream.write(getSize());
        outputStream.write(commandChar);
        writeBody(outputStream);
        outputStream.write(footer);
    }
}
```
여기서 Command 클래스에 writeBody의 추상 메소드를 도입해야 하는 점에 유의하자.
![KakaoTalk_Photo_2023-03-25-21-19-10](https://user-images.githubusercontent.com/60125719/227716965-82152587-600e-48d1-a169-e9413a29abb7.jpeg)

```Java
import java.io.OutputStream;

public class LoginCommand extends Command {

    private String userName;
    private String passwd;

    public LoginCommand(String userName, String passwd) {
        this.userName = userName;
        this.passwd = passwd;
    }

    protected char [] getCommandChar() {
        return new Char [] {0x01};
    }

    private int getSize() {
        return header.length + SIZE_LENGTH + CMD_BYTE_LENGTH + 
            footer.length + userName.getBytes().length + 1 + 
            passwd.getBytes().length + 1;
    }

}


import java.io.OutputStream;

public class AddEmplyoyeeCmd extends Command {

    String name;
    String address;
    String city;
    String state;
    String yearlySalary;

    private int getSize() {
        return header.length + 
            SIZE_LENGTH + 
            CMD_BYTE_LENGTH +
            footer.length +
            name.getBytes().length + 1 +
            address.getBytes().length + 1 +
            city.getBytes().length + 1 +
            state.getBytes().length + 1 +
            yearlySalary.getBytes().length + 1;
    }

    public AddEmplyoyeeCmd(String name, String address, String city, String state, int yearlySalary) {
        this.name = name;
        this.address = address;
        this.city = city;
        this.state = state;
        this.yearlySalary = Integer.toString(yearlySalary);
    }

    protected char [] getCommandChar() {
        return new Char [] {0x02};
    }

}
```

getSize 메소드를 보면..
```Java
private int getSize() {
    return header.length + SIZE_LENGTH + CMD_BYTE_LENGTH + 
        footer.length + userName.getBytes().length + 1 + 
        passwd.getBytes().length + 1;
}

private int getSize() {
    return header.length + 
        SIZE_LENGTH + 
        CMD_BYTE_LENGTH +
        footer.length +
        name.getBytes().length + 1 +
        address.getBytes().length + 1 +
        city.getBytes().length + 1 +
        state.getBytes().length + 1 +
        yearlySalary.getBytes().length + 1;
}
```
> 둘 다 헤더, 크기, 명령어 바이트의 길이, 꼬리말의 길이를 더하고 있다. 그리고 각 필드의 크기를 추가하고 있다. 계산 방법이 다른 부분, 즉 필드 크기 부분을 추출해보자.

```Java
private int getSize() {
    return header.length + SIZE_LENGTH
        + CMD_BYTE_LENGTH + footer.length + getBodySize();
}
```
> 이렇게 하면 두 개의 메소드는 동일한 코드를 가지게 된다.
![KakaoTalk_Photo_2023-03-25-21-25-15](https://user-images.githubusercontent.com/60125719/227717207-f1c58496-2aea-4d25-b295-077a8143e0ba.jpeg)

```Java
// AddEmployeeCmd 클래스의 getBody

protected int getBodySize() {
    return name.getBytes().length + 1 +
        address.getBytes().length + 1 +
        city.getBytes().length + 1 +
        state.getBytes().length + 1 +
        yearlySalary.getBytes().length + 1;
}

```
좀 더 개선해보면..

```Java
// AddEmployeeCmd 클래스의 getBody

protected int getFieldSize(String field) {
    return field.getBytes().length + 1;
}

protected int getBodySize() {
    return getFieldSize(name) +
        getFieldSize(address) +
        getFieldSize(city) +
        getFieldSize(state) +
        getFieldSize(yearlySalary);
}

```
getFieldSize 메소드를 Command 클래스로 끌어올리면 LoginCommand의 getBodySize 메소드에서도 사용할 수 있다.
```Java
protected int getBodySize() {
    return getFieldSize(name) + getFieldSize(password);
}
```

LoginCommand와 AddEmployeeCmd 둘 다 매개변수 목록을 받아와서 그 크기를 얻은 후 기록하는 일을 수행한다. commandChar변수를 제외하면 중복 부분을 일반화해 제거할 수 있다.  
기초클래스에서 List 타입의 변수를 선언하고 각 서브클래스의 생성자에서 필드를 추가해보자.

```Java

public class LoginCommand extends Command {

    // private String userName;
    // private String passwd;
    private List = fields = new ArrayList();

    public LoginCommand(String userName, String passwd) {
        // this.userName = userName;
        // this.passwd = passwd;
        this.fields.add(userName);
        this.fields.add(passwd)
    }

    protected char [] getCommandChar() {
        return new Char [] {0x01};
    }

    // private int getSize() {
    //     return header.length + SIZE_LENGTH + CMD_BYTE_LENGTH + 
    //         footer.length + userName.getBytes().length + 1 + 
    //         passwd.getBytes().length + 1;
    // }
    int getBodySize() {
        int result = 0;
        for (Iterator it = fields.iterator(); it.hasNext(); ) {
            String field = (String)it.next();
            result += getFieldSize(fields);
        }
        return result;
    }

    void writeBody(OutputStream outputStream) { // 얘도 개선 가능
        for (Iterator it = fields.iterator(); it.hasNext(); ) {
            String field = (String)it.next();
            writeField(outputStream, field);
        }
    }

}

```

이 구대의 메소드를 슈퍼클래스로 끌어올릴 수 있다.

```Java
public class Command {
    // protected static final byte[] header = {(byte)0xde, (byte)0xad};
    // protected static final byte[] footer = {(byte)0xb2, (byte)0xef};
    // protected static final int SIZE_LENGTH = 1;
    // protected static final int CMD_BYTE_LENGTH = 1;
    private static final byte[] header = {(byte)0xde, (byte)0xad};
    private static final byte[] footer = {(byte)0xb2, (byte)0xef};
    private static final int SIZE_LENGTH = 1;
    private static final int CMD_BYTE_LENGTH = 1;

    protected List = fields = new ArrayList();

    protected abstract char [] getCommandChar();

    // protected abstract void writeBody(OutputStream outputStream);
    void writeBody(OutputStream outputStream) {
        for (Iterator it = fields.iterator(); it.hasNext(); ) {
            String field = (String)it.next();
            writeField(outputStream, field);
        }
    }

    protected int getFieldSize(String field) {
        return field.getBytes().length + 1;
    }

    private int getBodySize() {
        int result = 0;
        for (Iterator it = fields.iterator(); it.hasNext(); ) {
            String field = (String)it.next();
            result += getFieldSize(fields);
        }
        return result;
    }

    private int getSize() {
        return header.length + SIZE_LENGTH
            + CMD_BYTE_LENGTH + footer.length + getBodySize();
    }


    // protected void writeField(OutputStream outputStream, String field) {
    private void writeField(OutputStream outputStream, String field) {
        outputStream.write(field.getBytes());
        outputStream.write(0x00);
    }

    public void write(OutputStream outputStream) throws Exception {
        outputStream.write(header);
        outputStream.write(getSize());
        outputStream.write(commandChar);
        writeBody(outputStream);
        outputStream.write(footer);
    }
}

public class LoginCommand extends Command {

    // private String userName;
    // private String passwd;

    public LoginCommand(String userName, String passwd) {
        // this.userName = userName;
        // this.passwd = passwd;
        fields.add(userName);
        fields.add(passwd);
    }

    protected char [] getCommandChar() {
        return new Char [] {0x01};
    }

    // private int getSize() {
    //     return header.length + SIZE_LENGTH + CMD_BYTE_LENGTH + 
    //         footer.length + userName.getBytes().length + 1 + 
    //         passwd.getBytes().length + 1;
    // }

}

public class AddEmplyoyeeCmd extends Command {

    // String name;
    // String address;
    // String city;
    // String state;
    // String yearlySalary;

    // private int getSize() {
    //     return header.length + 
    //         SIZE_LENGTH + 
    //         CMD_BYTE_LENGTH +
    //         footer.length +
    //         name.getBytes().length + 1 +
    //         address.getBytes().length + 1 +
    //         city.getBytes().length + 1 +
    //         state.getBytes().length + 1 +
    //         yearlySalary.getBytes().length + 1;
    // }

    public AddEmplyoyeeCmd(String name, String address, String city, String state, int yearlySalary) {
        // this.name = name;
        // this.address = address;
        // this.city = city;
        // this.state = state;
        // this.yearlySalary = Integer.toString(yearlySalary);
        fields.add(name);
        fields.add(address);
        fields.add(city);
        fields.add(state);
        fields.add(Integer.toString(yearlySalary));
    }

    protected char [] getCommandChar() {
        return new Char [] {0x02};
    }

}
```

![KakaoTalk_Photo_2023-03-25-21-43-01](https://user-images.githubusercontent.com/60125719/227717935-a9296ceb-6325-4466-ab9c-95e0484cff31.jpeg)


이제끝? 마지막 확인. 애초에 명령어들이 클래스로 분리되어있는게 맞나?
```Java
List arguments = new ArrayList();
arguments.add("Mike");
arguments.add("asdsad");
Command.send(stream, 0x01, arguments);
```
> 호출하는쪽에 부담. 사용자가 이를 관리해야 하기 때문. 
```Java
Command.sendAddEmployee(stream, "Mike", "122 Elm St", "Maami", "FL", 10000);
Command.SendLogin(stream, "Mike", "asdsad");
```
> 클라이언트 코드를 다 바꿔야함
  
-> 어쨌든 클래스가 맞는듯 이라는 결론!  
이제 진짜 끝? ㄴㄴ cmd -> command로 클래스 이름을 바꿔주자.
















