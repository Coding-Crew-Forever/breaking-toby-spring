# 관심사의 분리

관심이 같은 것끼리 모으고, 관심이 다른 것은 떨어져 있게 하는 것

## **1. 중복 코드 메서드 분리**

함수마다 커넥션을 가져오는 중복된 코드가 있다면, 주복된 DB 연결 코드를 따로 함수로 분리시켜 메소드를 호출하는 방식으로 구현한다.

## **2. 상하위 클래스 사용(상속)**

추가로 소스 코드를 고객사에 제공하지 않고 고객에 맞게 스스로 DB 커넥션 생성을 할 수 있게 한다면?

UserDao에서 메소드의 구현 코드를 제거하고 getConnection을 추상 메소드로 만든다. 구현은 각 고객사에서 UserDao를 상속받아 getConnection을 구현하면 된다.

템플릿 메소드 패턴

- 슈퍼 클래스에 기본적인 로직을 만들고 그 기능의 일부를 추상 메소드나 오버라이딩이 가능한 protected 메소드 등으로 만든 뒤 서브 클래스에서 이런 메소드를 필요에 맞게 구현해서 사용하도록 하는 방법

팩토리 메소드 패턴

- 서브 클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 것

```java
public abstract class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
	}
	
	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
	}
	
	public abstract Connection getConnection() throws ClassNotFoundException, SQLExcecption;
}
```

```java
public class NUserDao extends UserDao {
	public Connection getConnection() throws ClassNotFoundException, SQLException {
		// N사 DB Connection 생성 코드
	}
}

public class DUserDao extends UserDao {
	public Connection getConnection() throws ClassNotFoundException, SQLException {
			// D사 DB Connection 생성 코드
	}
}
```

## 3. 클래스 분리

관심사가 다르고 변화의 성격이 다른 코드는 클래스를 분리시킨다.

```java
public class UserDao {
	private SimpleConnectionMaker simpleConnectionMaker;
	
	public UserDao() {
		simpleConnectionMaker = new SimpleConnectionMaker();
	}
	
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = simpleConnectionMaker.makeNewConnection();
	}
	
	public User get(String id) throws ClassNotFoundException, SQLException }
		Connection c = simpleConnectionMaker.makeNewConnection();
	}
```

```java
public class SimpleConnectionMaker {
	public Connection MakeNewConnection() throws ClassNotFoundException, SQLException {
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "spring", "book");
		
		return c;	
	}
}
```

## 4. 인터페이스 사용

### 기존 코드 문제

1. 고객사마다 커넥션 제공 클래스의 함수가 다르다면 UserDao에서 해당 함수를 일일이 수정해야 한다.
2. DB 커넥션을 제공하는 클래스가 어떤 것인지를 UserDao가 구체적으로 알고 있어야 한다. UserDao에 DB 클래스의 인스턴스 변수까지 정의하니 고객사에서 다른 클래스를 구현하면 UserDao를 수정해야 한다.

→ 즉, UserDao가 바뀔 수 있는 정보(DB 커넥션을 가져오는 클래스)에 대해 너무 많이 알고 있다.

이는 중간에 인터페이스를 사용하면 된다. 

(UserDao - Connection 인터페이스 - 고객사 연결 클래스) 

```java
public interface ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```

```java
public class DConnectionMaker implements ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException {
	}
}
```

```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao() {
		connectionMaker = new DConnectionMaker(); // 구현 클래스를 알아야 하는 문제 발생
	}
	
	public void add(User user) throws ClassNotFoundException, SQLException{
		Connection c = connectionMaker.makeConnection();
	}
	
	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = connectionMaker.makeConnection();
	}
}
```

## 5. 관계 설정 책임 분리

기존 코드에서 아직 UserDao에 어떤 ConnectionMaker 구현 클래스를 사용할 지 결정하는 코드가 남아있다.

UserDao가 어떤 특정 구현 클래스 사이의 관계를 설정해주는 것에 대해 분리해야 한다.

UserDao의 모든 코드는 ConnectionMaker 인터페이스 외에 어떤 클래스와도 관계를 가져서 안된다.

```java
public UserDao(ConnectionMaker connectionMaker) {
	this.connectionMaker = connectionMaker;
}
```

```java
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ConnectionMaker connectionMaker = new DConnectionMaker(); // 고객사에 따라 변경
		UserDao dao = new UserDao(connectionMaker);
	}
}
```

UserDaoTest는 UserDao와 ConnectionMaker 구현 클래스와의 런타임 오브젝트 의존 관계를 설정하는 책임을 담당하게 된다.

그래서 특정 ConnectionMaker 구현 클래스의 오브젝트를 만들고 UserDao 생성자 파라미터에 넣어 두개의 오브젝트를 연결해준다.

*UserDaoTest → UserDao → ConnectionMaker ← DConnectionMaker ← UserDaoTest*

위 구조는 클래스나 모듈은 확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다는 개방 폐쇄 원칙(OCP)을 잘 보여주는 예시이다.

UserDao는 DB 연결 방법 기능 확장에는 개방되어 있고, 인터페이스로 변화가 불필요하게 일어나지 않도록 닫혀있다.

### 높은 응집도

변화가 일어날 때 해당 모듈에서 변하는 부분이 크다.

변경이 일어날 때 모듈의 많은 부분이 함께 바뀐다면 응집도가 높은 것

### 낮은 결합도

책임과 관심사가 다르오브젝트 또는 모듈과는 느슨하게 연결된 형태를 유지하는 것

관계는 유지하는 데 꼭 필요한 최소한의 방법만 간접적인 형태로 제공하고, 나머지는 서로 독립적이고 알 필요도 없게 만들어주는 것

### 전략 패턴

자신의 기능 컨텍스트에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴

위 UserDaoTest - UserDao - ConnectionMaker 구조에서 UserDao는 전략 패턴의 컨텍스트에 해당되며 DB 연결 방식을 ConnectionMaker라는 인터페이스로 분리하여 이를 구현한 클래스(전략)을 바꿔가며 사용할 수 있도록 하였다.

# 제어의 역전(IOC)

### 팩토리

클래스의 역할은 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 것인데, 이런 일을 하는 오브젝트를 팩토리라고 부른다.

기존 UserDaoTest에서 테스트라는 책임에 다른 책임을 주었으니, DaoFactory 클래스를 사용해 UserDao와 Connection을 생성하도록 관심사 분리를 한다.

```java
public class DaoFactory {
	public UserDao userDao() {
		ConnectionMaker connectionMaker = new DConnectionMaker();
		UserDao userDao = new UserDao(connectionMaker);
		return userDao;
	}
}
```

```java
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		UserDao dao = new DaoFactory().userDao();
	}
}
```

위와 같이 구성하면 고객사에 UserDao를 공급할 때 UserDao, ConnectionMaker, DaoFactory(소스코드)를 제공하여 고객사마다 변경이 필요하면 DaoFactory 코드를 수정하면 되므로, UserDao는 변경할 필요가 없게 된다.

이렇게 분리하게 되면, 애플리케이션의 컴포넌트 역할을 하는 오브젝트와 애플리케이션의 구조를 결정하는 오브젝트가 분리된다는 장점이 있다. (컴포넌트 - UserDao, ConnectionMaker / 구조 결정 - DaoFactory)

### 제어의 역전

모든 제어 권한을 자신이 아닌 다른 대상에게 위임하여 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지 않고, 생성하지 않으며, 자신도 어떻게 생성되고 사용되는지 모른다.

예를 들어, 서블릿도 개발자가 직접 제어할 수 있는 방법은 없다. 대신 서블릿에 대한 제어 권한을 가진 컨테이너가 적절한 시점에 서블릿 클래스의 오브젝트를 만들고 그 안의 메소드를 호출한다. 이렇게 서블릿이나 JSP, EJB처럼 컨테이너 안에서 동작하는 구조는 제어의 역전 개념이 적용되어 있다.

## 프레임워크

프레임워크도 제어의 역전 개념이 적용된 대표적인 기술이다.

### 라이브러리와 프레임워크

라이브러리를 사용하는 애플리케이션 코드는 애플리케이션 흐름을 직접 제어하며, 동작하는 중에 필요한 기능이 있을 때 능동적으로 라이브러리를 사용한다.

반대로, 프레임워크는 애플리케이션 코드는 프레임워크가 짜놓은 틀에서 수동적으로 동작해야 한다.

프레임워크 위에 개발한 클래스를 등록해두고, 프레임 워크가 흐름을 주도하는 중에 개발자가 만든 애플리케이션 코드를 사용하도록 만드는 방식이다.

---

# 스프링 IoC

## 빈

스프링이 IoC 방식으로 관리하는 오브젝트

스프링이 직접 생성과 제어를 담당하는 오브젝트만을 빈이라고 한다.

## 빈 팩토리

빈의 생성과 관계 설정과 같은 제어를 담당하는 IoC 오브젝트

빈을 등록, 생성, 조회, 반환 등 부가적인 빈을 관리하는 기능을 담당

## 애플리케이션 컨텍스트

빈 팩토리를 확장한 IoC 컨테이너

스프링에서 IoC 컨테이너, 스프링 컨테이너, 빈 팩토리 등으로 불리기도 한다.

스프링에서는 보통 빈 팩토리보다는 이를 좀 더 확장한 애플리케이션 컨텍스트를 주로 사용한다.

빈 팩토리에 기능에 추가로 좀 더 애플리케이션 전반에 걸쳐 모든 구성요소의 제어 작업을 담당한다.

예를 들어, 애플리케이션에서 IoC를 적용해서 관리할 모든 오브젝트에 대한 생성과 관계 설정을 담당한다.

대신 직접 오브젝트를 생성하고 관계를 맺는 코드는 없고, 별도의 설정 정보를 통해 얻거나 외부 오브젝트 팩토리에 작업을 위임하고 결과를 가져다가 사용한다. 

client가 userDao를 요청하면 ApplicationContext에서 getBean으로 userDao를 조회하고 @Configuration이 붙은 DaoFactory에 가서 userDao 빈을 찾아 전달해준다.

### 장점

1. **클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다**
    
    클라이언트가 필요한 오브젝트를 가져오려면 어떤 팩토리 클래스를 사용해야 하는지 알아야 하고, 필요할 때마다 팩토리 오브젝트를 생성할 필요가 없다.
    
2. **애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다.**
    
    오브젝트 생성, 다른 오브젝트와 관계 설정 외에도 오브젝트 생성에서 방식, 시점과 전략을 다르게 가져갈 수 있고, 자동생성, 오브젝트에 대한 후처리, 정보의 조합, 설정 방식의 다변화, 인터셉팅 등 다양한 기능을 제공한다.
    
    빈이 사용할 수 있는 기반 기술 서비스나 외부 시스템과 연동 등을 컨테이너 차원에서 제공해준다.
    
3. **애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다.**
    
    getBean()은 빈의 이름을 통해 빈을 찾는 방법이고, 타입만으로 빈을 검색하거나, 특별한 애노테이션 설정이 된 빈을 찾을 수도 있다.
    

### @Configuration

스프링 설정 정보로 인식될 수 있도록 하는 어노테이션

스프링이 빈 팩토리를 위한 오브젝트 설정을 담당하는 클래스라고 인식할 수 있도록 DaoFactory에 붙여준다.

<aside>

### 스프링 설정 정보

애플리케이션 컨텍스트, 빈 팩토리가 IoC를 적용하기 위해 사용하는 메타 정보

</aside>

### @Bean

오브젝트를 만들어주는 메소드에 붙여준다.

위에서는 UserDao와 ConnectionMaker를 생성하는 메소드에 붙여준다.

빈 어노테이션이 붙은 메소드는 메소드명이 빈 이름이 된다.

```java
@Configuration
public class DaoFactory {
	@Bean
	public UserDao userDao(){
		return new UserDao(connectionMaker());
	}
	
	@Bean
	public ConnectionMaker connectionMaker() {
		return new DConnectionMaker();
	}
}
```

위 DaoFactory를 설정 정보로 사용하는 애플리케이션 컨텍스트를 하려면 아래와 같이 사용하면 된다.

AnnotationConfigApplicationContext는 Configuration이 붙은 코드를 설정 정보로 사용할 수 있고, 생성자 파라미터로 해당 어노테이션이 붙은 클래스를 넣어준다.

이렇게 준비된 ApplicationContext의 getBean이라는 메소드로 UserDao의 오브젝트를 가져올 수 있다.

```java
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ApplicationContext context = new AnnotationConfigAppliactionContext(DaoFactory.class);
		UserDao dao = context.getBean("userDao", UserDao.class);
	}
}
```

---

## 동일성과 동등성

### 동일성

완전히 같은 동일한 오브젝트 (== 연산자)

### 동등성

동일한 정보를 담고 있는 오브젝트 (equals())

동일한 오브젝트는 동등하기도 하다. 하지만 동등한 오브젝트는 항상 동일하지는 않다.

## 스프링 애플리케이션 컨텍스트에서 여러번 빈을 생성한다면?

```java
ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);

UserDao dao3 = context.getBean("userDao", UserDao.class);
UserDao dao4 = context.getBean("userDao", UserDao.class);

System.out.println(dao3);
System.out.println(dao4);
```

애플리케이션 컨텍스트를 사용하여 getBean으로 가져온 두 오브젝트는 동일하다.

(사용하지 않고 그냥 두번 호출하면 두 오브젝트는 동일하지 않다.)

스프링은 기본적으로 내부에서 생성하는 빈 오브젝트를 모두 싱글톤으로 만든다.

애플리케이션 컨텍스트는 싱그론을 저장하고 관리하는 싱글톤 레지스트리이다.

## 싱글톤?

어떤 클래스를 애플리케이션 내에서 제한된 인스턴스 개수, 이름처럼 주로 하나만 존재하도록 강제하는 패턴이다.

하나만 만들어지는 클래스의 오브젝트는 애플리케이션 내에서 전역적으로 접근이 가능하다.

애플리케이션의 여러 곳에서 공유하는 경우에 주로 사용한다.

서버 환경에서는 서비스 싱글톤의 사용이 권장되지만, 사용하기 까다로워 조심해서 사용해야 한다.

## 왜 스프링에서 싱글톤?

매번 클라이언트에서 요청이 올 때마다 각 로직을 담당하는 오브젝트를 새로 만들어 사용하면 부하가 걸리기 쉽다.

그래서 서블릿은 대부분 멀티 스레드 환경에서 싱글톤으로 동작하고, 

서블릿 클래스 하나당 하나의 오브젝트만 만들어두고, 사용자의 요청을 담당하는 여러 스레드에서 하나의 오브젝트를 공유해 동시에 사용한다.

## 싱글톤 구현

1. 클래스 밖에서는 오브젝트를 생성하지 못하도록 생성자를 private으로 만든다.
2. 생성된 싱글톤 오브젝트를 저장할 수 있는 자신과 같은 타입의 static 필드를 정의한다.
3. static 팩토리 메소드인 getInstance()를 만들고, 이 메소드가 최초로 호출되는 시점에서 한번만 오브젝트가 만들어지게 한다. 생성된 오브젝트는 스태틱 필드에 저장된다. 또는 스태틱 필드의 초기값으로 오브젝트를 미리 만들어둘 수도 있다.
4. 한번 오브젝트가 만들어지고 난 후에는 getInstance() 메소드를 통해 이미 만들어져 static 필드에 저장해둔 오브젝트를 넘겨준다.

```java
public class UserDao {
	private static UserDao INSTANCE;
	
	private UserDao(ConnectionMaker connectionMaker){
		this.connectionMaker = connectionMaker;
	}
	
	public static synchronized UserDao getInstance() {
		if(INSTANCE == null) INSTANCE = new UserDao(???);
		return INSTANCE;
	}
}
```

## 싱글톤 한계

1. private 생성자를 갖고 있기 때문에 상속할 수 없다.
2. 테스트 하기 어렵다.
3. 서버 환경에서 싱글톤이 하나만 만들어지는 것을 보장하지 못한다.
    
    여러 개의 JVM에 분산 되어서 설치 되는 경우에 각각 독립적으로 오브젝트가 생기기 때문에 싱글톤으로서 가치가 떨어진다.
    
4. 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다.
    
    아무 객체나 자유롭게 접근하고 수정하고 공유할 수 있는 전역 상태를 갖는 것은 객체지향에서 권장하지 않는다. 
    

## 싱글톤 레지스트리

스프링에서는 싱글톤을 권장하지만, 자바로 싱글톤을 구현하기에는 한계가 있기 때문에

스프링에서 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공하는데, 이를 싱글톤 레지스트리라고 한다.

static 메소드와 private 생성자를 사용해야 하는 클래스가 아니라, 평범한 자바 클래스도 IoC 방식의 컨테이너를 사용해서 생성, 관계 설정, 사용 등 제어권을 컨테이너에게 넘겨 싱글톤으로 만들어져 관리되게 할 수 있다.

싱글톤 레지스트리 덕분에 싱글톤 방식으로 사용될 애플리케이션 클래스라도 public 생성자를 가질 수 있다.

그래서 다양한 디자인 패턴을 적용하는 데 제약이 없다.

이렇듯 스프링은 IoC 컨테이너일 뿐만 아니라, 싱글톤 레지스트리이다.

## 싱글톤과 오브젝트의 상태

싱글톤은 멀티스레드 환경이라면 여러 스레드가 동시에 접근해서 사용할 수 있으므로 상태 관리에 신경 써야 한다.

기본적으로 싱글톤은 상태 정보를 내부에 갖고 있지 않은 stateless 방식으로 만들어져야 한다.

다중 사용자의 요청을 한번에 처리하는 스레드들이 동시에 싱글톤 오브젝트의 인스턴스 변수를 수정하는 것은 매우 위험하다.

따라서 싱글톤은 인스턴스 필드의 값을 변경하고 유지하는 상태 유지(stateful) 방식으로 만들지 않는다.

### 문제 되는 예시

```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	private Connection c;
	private User user;
	
	public User get(String id) throws ClassNotFoundException, SQLException {
		this.c = connectionMaker.makeConnection();
		this.user = new User();
		this.user.setId(rs.getString("id"));
		
		return this.user;
	}
}
```

위와 같이 설정한다면 멀티스레드 환경에서 문제가 발생한다.

Connection과 User 클래스는 매번 새로운 값으로 바뀌는 정보를 담는데, 이를 인스턴스 변수로 선언해서 큰 문제가 발생한다.

이렇듯 개별적으로 바뀌는 정보는 로컬 변수로 정의하거나, 파라미터로 주고 받으면서 사용해야 한다.

반면 ConnectionMaker는 읽기 전용 정보이고, 한 번 초기화 하면 이후에 수정하지 않아도 되기 때문에 인스턴스 변수로 사용해도 된다.

## 빈 스코프

빈이 생성되고, 존재하고, 적용되는 범위

스프링 빈의 기본 스코프는 싱글톤이다.

싱글톤 스코프는 컨테이너 내에 한 개의 오브젝트만 만들어져서, 강제로 제거하지 않는 한 스프링 컨테이너가 존재하는 동안 계속 유지된다.

그 외)

프로토타입 스코프는 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트를 만들어준다.  

요청 스코프는 웹을 통해 새로운 HTTP 요청이 생길 때마다 생성된다.

세션 스코프는 웹의 세션과 스코프가 유사하다.

---

# 의존 관계 주입(DI)

스프링이 제공하는 IoC 방식을 의존 관계 주입이라고 부른다.

IoC 컨테이너 = DI 컨테이너

## 의존 관계란?

A가 B에 의존하고 있을 때 B가 변하면 A에 영향을 미친다.

의존 관계에는 방향성이 있다.

### 모델링 시점의 의존 관계

위에서 설정한 UserDao - ConnectionMaker - DConnectionMaker 관계에서

UserDao는 ConnectionMaker 인터페이스에만 의존하므로, ConnectionMaker가 변경되면 영향을 받지만 이를 구현한 DConnectionMaker가 변경된다면 UserDao는 영향을 받지 않는다.

이렇듯 인터페이스에 대해서만 의존 관계를 만들어두면 느슨한 결합이 되어 변경에 자유로워 진다.

### 런타임 의존 관계

설계 시점의 의존관계가 실체화 된 것.

### 의존 관계 주입

의존 관계 주입은 아래의 조건을 충족하는 작업을 말한다.

- 클래스 모델이나 코드에는 런타임 시점의 의존 관계가 드러나지 않는다. 그러기 위해서는 인터페이스에만 의존하고 있어야 한다.
- 런타임 시점의 의존 관계는 컨테이너나 팩토리 같은 제 3의 존재가 결정한다.
- 의존 관계는 사용할 오브젝트에 대한 래퍼런스를 외부에서 제공해줌으로써 만들어진다.

핵심은 관계 설정을 해주는 제 3의 존재이며, 이는 위에서 만든 DaoFactory라고 할 수 있다.

```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
}
```

DI 컨테이너(DaoFactory)는 UserDao를 만드는 시점에서 생성자 파라미터로 이미 만들어진 DConnectionMaker의 오브젝트를 전달한다.

외부에서 내부로 무엇인가 넘겨주므로 생성자를 활용한 의존 관계 주입에 관한 코드이다.

이렇게 생성자 파라미터를 통해 전달받은 런타임 의존관계를 갖는 오브젝트는 인스턴스 변수에 저장해둔다.

### 의존 관계 검색

의존 관계를 맺는 방법이 외부로부터 주입이 아니라 스스로 검색을 이용해 자신이 필요로 하는 의존 오브젝트를 능동적으로 찾는다.

자신이 어떤 클래스의 오브젝트를 이용할 지 결정하는 작업은 외부 컨테이너에 IoC로 맡기지만, 이를 가져올 때는 메소드나 생성자 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용한다.

```java
public UserDao() {
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
	this.conntectionMaker = context.getBean("connectionMaker", ConnectionMaker.class);
}
```

### 의존 관계 주입과 의존 관계 검색(DI와 DL)

의존 관계 주입 방식이 더 코드가 단순하고 깔끔해서 좋다.

하지만 검색 방식을 사용해야 할 때가 있다.

테스트 코드에서 의존 관계 검색 방식인 getBean()을 사용했다면 IoC와 DI 컨테이너를 적용했다 하더라도 애플리케이션 기동 시점에서 적어도 한 번은 의존 관계 검색 방식으로 오브젝트를 가져와야 한다.

static 메소드인 main에서는 DI를 이용해 오브젝트를 주입받을 방법이 없기 때문.

서버에서도 main 메소드와 비슷한 역할을 하는 서블릿에서 스프링 컨테이너에 담긴 오브젝트를 사용하려면 한번은 의존 관계 검색 방식을 사용해 오브젝트를 가져와야 한다.

→ 이미 스프링에서 구현 되어있게 때문에 직접 구현할 필요는 없다.

의존 관계 검색 방식에서는 검색하는 오브젝트는 자신이 스프링 빈일 필요가 없지만,

의존 관계 주입 방식에서는 DI가 적용되려면 두 오브젝트 모두 컨테이너가 만드는 빈 오브젝트여야 한다.

→ 먼저 자기 자신이 컨테이너가 관리하는 빈이 되어야 한다는 뜻이다.

### DI의 장점

런타임 클래스에 대한 의존 관계가 나타나지 않고, 인터페이스를 통해 결합도가 낮은 코드를 만드므로 다른 책임을 가진 사용 의존 관계에 있는 대상이 바뀌거나 변경되더라도 자신은 영향을 받지 않으며 변경을 통한 다양한 확장 방법에 자유롭다.

### 메소드를 통한 의존 관계 주입

1. 수정자 메소드(setter)를 이용한 주입
2. 일반 메소드를 이용한 주입
    
    1번에서 set과 하나의 파라미터만 사용해야 하는 제약이 싫다면, 여러 개의 파라미터를 갖는 일반 메소드를 DI 용으로 사용할 수 있다. 
    

```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public void setConnectionMaker(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
}
```

기존 생성자는 제거하고 대신 setConnectionMaker라는 메소드를 하나 추가

```java
// DaoFactory 수정자 메소드 DI를 사용하는 팩토리 메소드
@Bean
public UserDao userDao() {
	UserDao userDao = new UserDao();
	userDao.setConnectionMaker(connectionMaker());
	return userDao;
}
```

---

# XML을 이용한 설정

DaoFactory와 같은 자바 클래스를 이용하는 것 외에도 다양한 방법을 통해 DI 의존 관계 설정 정보를 만들 수 있다.

|  | 자바 | XML |
| --- | --- | --- |
| 빈 설정 파일 | @Configuration | <beans> |
| 빈 이름 | @Bean methodName() | <bean id=”methodName” |
| 빈 클래스 | return new BeanClass(); | class=”a.b.c… BeanClass”> |

```java
@Bean
public ConnectionMaker connectionMaker() {
	return new DConnectionMaker();
}

@Bean
public UserDao userDao() {
	UserDao userDao = new UserDao();
	userDao.setConnectionMaker(connectionMaker());
	return userDao;
}
```

```xml
<beans>
	<bean id="connectionMaker1" class="springbook...DConnectionMaker" />
	<bean id="UserDao" class="springbook.dao.UserDao">
		<property name="connectionMaker" ref="connectionMaker1" />
	</bean>
</beans>
```

## <Property name=? ref=? />

<property>는 name과 ref라는 두개의 애트리뷰트를 갖는다.

name은 DI에 사용할 수정자 메소드의 프로퍼티 이름

ref는 주입할 오브젝트를 정의한 빈 ID

## XML을 사용하는 애플리케이션 컨텍스트

XML에서 빈의 의존 관계 정보를 이용하는 IoC/DI 작업에는 GenericXmlApplicationContext를 이용하므로, 생성자 파라미터르 XML 파일 클래스 패스를 지정해주면 된다.

XML 파일은 클래스 패스 최상단에 두도록 하며, 이름은 applicationContext.xml이라고 짓는다.

```java
ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
```

## DataSource 인터페이스

DB 커넥션을 가져오는 오브젝트의 기능을 추상화해서 비슷한 용도로 사용할 수 있게 만든 인터페이스

ConnectionMaker와 같은 인터페이스를 사용하지 않고 DataSource를 사용해 getConnection()으로 DB 커넥션을 가져올 수 있다.

```java
import javax.sql.DataSource;

public class UserDao {
	private DataSource dataSource;
	
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	public void add(User user) throws SQLException {
		Connection c = dataSource.getConnection();
	}
}
```

```java
// DaoFactory
@Bean
public DataSource dataSource() {
	SimpleDriverDataSource dataSource = new SimpleDriverDataSource();
	
	dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
	dataSource.setUrl("");
	dataSource.setUsername("");
	dataSource.setPAssword("");
	
	return dataSource;
}

@Bean
public UserDao userDao() {
	UserDao userDao = new UserDao();
	userDao.setDataSource(dataSource());
	return userDao;
}
```

```xml
<beans>
	<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
		<property name="driverClass" value="com.mysql.jdbc.Driver" />
		<property name="url" value="jdbc:mysql://localhost/springbook" />
		<property name="username" value="spring" />
		<property name="password" value="book" />
	</bean>
	
	<bean id="userDao class="springbook.user.dao.UserDao">
		<property name="dataSource" ref="dataSource" />
	</bean>
</beans>
```
