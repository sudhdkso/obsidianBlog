![](https://img.notionusercontent.com/s3/prod-files-secure%2F560fc466-27da-429f-ba6b-9df88144a59a%2Fadc1e71d-349a-4503-b7e8-a007344b16c3%2FUntitled.png/size/w=2000?exp=1728905458&sig=Knwr3SeZqAoHqkAql4uMiC8LhD-drqrTqGAm_1rmxTo)


## PSA(Portable Service Abstraction)

- PSA란 **환경의 변화와 관계없이 일관된 방식의 기술로의 접근 환경을 제공하는 추상화 구조**를 말합니다**.**
    - **추상화 계층을 사용하여 어떤 기술을 내부에 숨기고 개발자에게 편의성을 제공해주는것이 서비스 추상화(Service Abstraction)**입니다.
- 이는 POJO 원칙을 철저히 따른 Spring의 기능으로 Spring에서 동작할 수 있는 Library들은 POJO원칙을 지키게끔 PSA형태의 추상화가 되어있음을 의미합니다.
- **PSA = 잘 만든 인터페이스**
- PSA가 적용된 코드라면 나의 코드가 바뀌지 않고, 다른 기술로 간편하게 바꿀 수 있도록 확장성이 좋고,
- Spring은**Spring Web MVC, Spring Transaction, Spring Cache 등의 다양한 PSA를 제공**합니다.

## PSA의 예시
###  1. Spring Web MVC

### **일반적인 서블릿의 형태**

- _Servlet을 사용하려면 HttpServlet을 상속 받고 doGet(), doPost() 등 오버라이딩하여 사용해야 합니다._
    
    ```java
    public class SpringServlet extends HttpServlet {
    	// GET
    	@Override
    	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    		super.doGet(req, resp);
    	}
    	
    	// POST
    	@Override
    	protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    		super.doPost(req, resp);
    	}
    }
    ```
    

####  Spring Web MVC의 형태

```java
@Controller
class UserController {
	@GetMapping("/api/user")
	@LogExecutionTime
	public String initCreationForm(Map<String, Object> model) {
		User user = new User();
		model.put("user", user);
		return VIEWS_USER_CREATE_OR_UPDATE_FORM;
	}

}
```

Spring Web MVC의 형태에서는 서블릿이 전혀 존재하지 않고 @Controller 어노테이션이 붙어있는 클래스에서 @GetMapping, @PostMapping과 같은 @RequestMapping 어노테이션을 사용해서 요청을 매핑합니다.

이는 실제로는 내부적으로 서블릿 기반으로 코드가 동작하지만 서블릿 기술은 추상화 계층에 의해 숨겨져 있는 것이기 때문입니다.

그래서 우리는 서블릿을 low level로 개발하지 않아도 됩니다.

###  2. Spring Transaction

### Low Level

```java
try (Connection conn = DriverManager.getConnection(
     "jdbc:coco://127.0.0.1:8080/test", "root", "password");
     Statement statement = conn.createStatement();) 
{
 
	 //start transaction block
	 conn.setAutoCommit(false); //default true
 
	 String SQL = "INSERT INTO Employees " +
	 			"VALUES (109, 24, '김일', '공장1')";

	 stmt.executeUpdate(SQL);
 
	 String SQL = "INSERTED INT Employees " +
	 			"VALUES (110, 26, '박이', '공장2')";

	 stmt.executeUpdate(SQL);
 
	 // end transaction block, commit changes
	 conn.commit();
 
	 // good practice to set it back to default true
	 conn.setAutoCommit(true);
 
} catch(SQLException e) {
	System.out.println(e.getMessage());
	conn.rollback();
 }
```

2번째 SQL문에 INSERTED INT 의 오타로 인해 커밋되지 않고 catch문으로 가게 되어 `conn.rollback();`으로 롤백 하는 코드입니다.

이처럼 Low Level로 트랜잭션을 처리하려면 명시적으로 `setAutoCommit()`과`commit()`, `rollback()`을 호출해야 합니다.

하지만 Spring이 제공하는`@Transactional`어노테이션을 사용하면 단순하게 메소드에 어노테이션을 붙여줌으로써

트랜잭션 처리가 간단하게 이루어집니다.

```java
@Transactional(readOnly = true)
Employees findById(Integer id);
```

이 또한 PSA로써 다양한 기술 스택으로 구현체를 바꿀 수 있습니다.

예를 들어 JDBC를 사용하는 DatasourceTransactionManager, JPA를 사용하는 JpaTransactionManager, Hibernate를 사용하는 HibernateTransactionManager를 유연하게 바꿔서 사용할 수 있다.

**기존 코드는 변경하지 않은 채로 트랜잭션을 실제로 처리하는 구현체를 사용 기술에 따라 바꿀 수 있는 것**입니다.

### 3. Spring Cache

Cache도 마찬가지로 JCacheManager, ConcurrentMapCacheManager, EhCacheCacheManager와 같은 여러가지 구현체를 사용할 수 있습니다.

사용자는**`@Cacheable`**어노테이션을 붙여줌으로써 구현체를 크게 신경쓰지 않아도 필요에 따라 바꿔 쓸 수 있습니다.

Spring 은 이렇게**특정 기술에 직접적 영향을 받지 않게** **객체를 POJO 기반으로 한번 씩 더 추상화**한 Layer 를 갖고 있으며 이를통해 일관성있는 서비스 추상화를 만들어냅니다.덕분에 코드는 더 견고해지고 기술이 바뀌어도 유연하게 대처할 수 있게 됩니다.

예를 들어 프로젝트를 하면서 캐시를 적용하게 되었습니다. 이 때 캐싱이 필요한 메소드에 **`@Cacheable`**이라는 어노테이션을 붙이게 됩니다. 그리고 JCacheManager를 구현체로 사용하게 되었습니다. 하지만 프로젝트를 하다 필요에 의해 EhCacheCacheManager로 구현체를 변경하게 되었습니다. 하지만 이 때 **`@Cacheable`** 이라는 어노테이션은 변경해줄 필요가 없습니다. 왜냐하면 Cache는 PSA이기 때문입니다. 구현체가 변경되었다고 하더라도 실제 코드는 변경하지 않아도 됩니다.

### [출처]

[https://dev-coco.tistory.com/83](https://dev-coco.tistory.com/83)

[https://dar0m.tistory.com/229](https://dar0m.tistory.com/229)