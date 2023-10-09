# Junit

### assertJ 새로운 메소드

- assertThat 에서 List의 size 검증 시 사용할 수 있는 `hasSize`

```java
assertThat(cafekiosk.getBeverages()).hasSize(2);
```

- assertThatThrownBy 로 실행하는 메소드의 예외 종류와 메시지까지 확인 가능

```java
assertThatThrownBy(() -> cafeKiosk.add(americano, 0))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("음료는 1잔 이상 주문하실 수 있습니다.");
```

### 테스트 하기 어려운 영역

- 외부 세계에 영향을 주는 코드
→ 로그나 메시지 발송(이메일,알림톡 등) DB에 기록하기
- 관측할 때마다 다른 값에 의존하는 코드
→ 현재 날자, 시간, 랜덤 값, 전역변수,함수, 사용자 입력 등
    
    ```java
    public Order createOrder() {
    				// LocalDateTime 은 메소드를 실행할 때 마다 유동적으로 변경된다.
            LocalDateTime currentDateTime = LocalDateTime.now();
            LocalTime currentTime = currentDateTime.toLocalTime();
            if (currentTime.isBefore(SHOP_OPEN_TIME) || currentTime.isAfter(SHOP_CLOSE_TIME)) {
                throw new IllegalArgumentException("주문 시간이 아닙니다. 관리자에게 문의하세요.");
            }
    
            return new Order(currentDateTime, beverages);
        }
    ```
    
    변경 후
    
    ```java
    public Order createOrder(LocalDateTime currentDateTime) {
            LocalTime currentTime = currentDateTime.toLocalTime();
            if (currentTime.isBefore(SHOP_OPEN_TIME) || currentTime.isAfter(SHOP_CLOSE_TIME)) {
                throw new IllegalArgumentException("주문 시간이 아닙니다. 관리자에게 문의하세요.");
            }
    
            return new Order(currentDateTime, beverages);
        }
    ```
    
    - 테스트 때문에 비즈니스 로직을 변경해야 하는것은 문제가 되는가 ? 
    → NO. 테스트를 실행할때마다 결과값이 변경되는 코드는 잘못된 코드일 가능성이 높다. 이러한 관심사를 잘 분리해야한다.
    - 즉, 외부세계에 있는 값에 메소드에 의존을 하고 있을 때 테스트 하기 어려워진다. 즉, 외부세계와 단절된 형태로 메소드를 만들자. (순수함수)

### AssertThat 을 좀 더 스마트하게 사용하기

```java
List<Product> products = productRepository.findAllBySellingStatusIn(list)

assertThat(products).hasSize(2)
				.extracting("id", "name" , "sellingStatus")
				.containExactlyInAnyOrder(
							tuple("001", "아메리카노", "SELL"),
							tuple("002", "라떼", "HOLD")
		);
```

- Extracting : products 의 필드들.
- containExactlyInAnyOrder : 필드의 값 들. AnyOrder 는 순서에 상관 없이.
- containExactly : 필드의 값 들. 순서도 테스트 패스 가능 여부에 포함됨.

### @DataJpaTest

- JPA 관련 Bean 만 가져와서 테스트.
- 테스트 과정에서 Insert (save) 가 있다면 Rollback 처리. (@Transactional 어노테이션을 추가한것과 동일한 효과)

### @AfterEach 클린징 처리

- deleteAllInBatch 의 영향력 확인하기. (테스트에서 추가한 내용만 지우는 것 인지..)

### @Transactional 는 테스트에서 지양

- 트랜잭션 어노테이션이 테스트코드에 포함되어 있으면 해당 테스트가 완료된 이후 자동으로 DB 변경 내용이 롤백된다.
- BUT, 지양해야 하는 이유는 바로 실제 비즈니스 로직에 트랜잭션 어노테이션이 누락되었을 때 테스트코드에 존재하는 트랜잭션 어노테이션으로 인해 테스트가 성공하기 때문이다.
→ 즉, 테스트에서 실제 비즈니스 로직의 실제를 걸러내지 못할 위험이 있다.

### Repository (Persistence) 테스트

는 save 로 데이터를 추가해주고 검증 후 삭제한다.

### Presentation Layer TEST

- `MockMvc` 를 사용해서 작성하는데, 이는 하위 레이어(비즈니스 또는 펄시스턴스)는 모두 mocking 처리를 해주는 것.

```java
// 테스트 하려는 컨트롤러의 클래스 이름을 테스트 클래스 위 상단의 아래 어노테이션과 함께 추가한다.
@WebMvcTest(controllers = ProductController.class)

// Mock 객체를 사용하기 위한 Mock 의존성 등록
@Autowired
private MockMvc mockMvc

// 하위 레이어를 의존성 주입 받는데, 가짜 객체(Mock)을 주입받는다.
@MockBean
private ProductService productService
```


<img width="645" alt="스크린샷 2023-10-09 오후 6 37 08" src="https://github.com/kihyuk-jeong/cafekiosk/assets/39195377/63c70dbb-833b-4999-80d1-0ac3e1986a0d">

**추가적으로, 어떠한 서비스 로직이 선 실행 되어야 하는 구조에서는 다음과 같이 `when` 메소드를 사용해서 비즈니스 레이어의 실행 결과를 mocking 한다.**

```java
//given
List<ProductResponse> result = List.of(.....)

when(productService.getSellingProducts()).thenRetrun(result);
```

### CQRS

- Command (writer 작업 즉, create, update, delete) 와 read 를 분리하자.
- @Transactional(readOnly=ture) 로 분리할 수 있다.
- 위 어노테이션을 사용하면 JPA 사용 시 스냅샷 저장을 하지 않고 변경감지 체크를 하지 않기 때문에 성능 향상이 있다.
- 즉 CQRS 패턴이란 쓰기와 읽기를 철저히 분리하는 패턴

### Moking

통합테스트에서 일부 기능한 Mock 객체로 등록해놓고 mocking 처리를 할 수 있다.
→ 예를들어, 이메일 발송 기능을 제공하는 Bean 이 포함된 테스트 경우 테스트가 진행될 때 마다 이메일이 실제로 발송되어야 한다. 이 때 메일 발송 Bean 을 mocking 처리 해서 테스트를 진행할 수 있다.

**(`any()` 메소드를 사용하고 내부에 type 을 명시)**
<img width="661" alt="스크린샷 2023-10-09 오후 6 35 28" src="https://github.com/kihyuk-jeong/cafekiosk/assets/39195377/5fbc9f37-da40-47dc-82dd-a642087c5967">


참고로, 메일 전송과 같은 외부 네트워크를 타는 긴 작업은 트랜잭션을 걸지 않는것이 좋다.
→ 트랜잭션을 걸면 커넥션 풀에서 커넥션을 하나 할당하고 이러한 외부 네트워크 작업은 오랜 시간동안 커넥션을 물고있어야 하기 때문이다.
<img width="700" alt="스크린샷 2023-10-09 오후 6 36 01" src="https://github.com/kihyuk-jeong/cafekiosk/assets/39195377/ab7bab1f-6de5-41ee-8ac9-5ce6150dfbdf">

### 어노테이션 별 차이점 (Mock)

- @MockBean : 스프링 통합테스트 시 mock 객체를 사용하기 위한 어노테이션으로, Bean 자체가 스프링 위에서 동작하므로 통합테스트(SpringBootTest) 시 mocking 처리를 하기 위해서는 해당 어노테이션을 사용해야 한다.
- @Mock : 통합테스트를 사용하지 않았을 경우 사용하는 어노테이션으로, 클래스 상단에 `@ExtendWith(MockitoExtension.class)` 를 추가하여 Mock 으로 객체를 만든다는것을 명시줘야한다.
- @Spy : Mock 객체의 여러가지 메소드 중 일부 메소드만 mocking 처리 하고 나머지는 실제 객체로 사용하고 싶을 때 사용하는 어노테이션. (반드시 **********************doReturn()********************** 을 사용해야 함.)
    
    ```java
    @Spy
    MailSendClient mailSendClient;
    
    ...
    
    @Test
    void test() {
    
    	doReturn(true)
    			.when(mailSendClient)
    			.sendEmail(anyString(), anyString());
    
    }
    ```
    

### Mock 에서 When() vs Given() ?

기존에는 `when()` 만 존재하다가 given() 절대 when 이라는 이름의 메소득 존재하는것이 어색해서 BDDMockito 에서 `given()` **메소드가 추가된 것. 기능은 동일하다.
→ `BDDMockito.given()` 을 사용하면 된다.**

### Test Fixture 정리

- 외부 로직 테스트는 Mocking  처리. (방법이 없음)
- Test Fixture 는 테스트를 위해 원하는 상태로 변경시킨 일련의 객체를 의미한다. 주로 given 절에서 만들어진다.
    - Test 시 `@BeforeEach` 부분에서 given 절에서 수행되는 test Fixture(테스트용 객체)를 생성하는 행위는 지양해야 한다. (문서로써의 역할을 하지 못한다. 위 아래로 왔다 갔다..)
    주로 테스트에 아무런 영향을 주지 않는 작업은 여기서 해도 좋다.
    

### @ParameterizedTest

동일한 테스트 이지만 값만 바꿔서 여러 번 테스트 해보고 싶을 때.

**AS-IS**
<img width="642" alt="스크린샷 2023-10-09 오후 6 34 49" src="https://github.com/kihyuk-jeong/cafekiosk/assets/39195377/c24ce7e8-4f79-4c62-a6b8-fddde9504e2d">  


**TO-BE**
<img width="628" alt="스크린샷 2023-10-09 오후 6 35 03" src="https://github.com/kihyuk-jeong/cafekiosk/assets/39195377/eb487340-dbab-4bcc-a41f-25521ecca5f3">


- @CsvSource 에 각각 파라미터를 구분자로 여러 개 추가할 수 있다.
- 위 어노테이션 외에도 `@MethodSource` 도 사용할 수 있다.
    
    ```java
    @MethodSource("temp")
    @ParameterizedTest
    void test(ProductType productType, boolean expected) {
    
    		boolean result = ProductType.containStockType(productType);
    
    		assertThat(result).isEqualTo(expected);
    
    }
    
    private static Stream<Arguments> temp() {
    	return Stream.of(
    				Arguments.of(ProductType,HANDMADE, false),
    				Arguments.of(ProductType,BOTTLE, false),
    				Arguments.of(ProductType,BAKERY, true)
    		);
    }
    ```
    

### 다이나믹 테스트

테스트를 할 때 다른 작업과 유기적으로 연결되어야 하는 경우 사용할 수 있다.

예를들어서, 재고가 1개인 상품에 대해 재고 차감 테스트를 하려고 할 때, 처음 재고 차감을 하게 되면 1개가 정상적으로 감소하게 될 것이고, 그 다음 바로 재고차감을 하게 되면 재고 부족 예외가 발생할 것이다.

이러한 상황에서 재고 차감을 연속적으로 하면서 첫 번째는 성공 두번째는 예외 처리와 같이 다이나믹하게 테스트를 작성할 수 있다.

- `@TestFactory` 를 사용한다.

```java
@DisplayName("재고를 주어진 개수만큼 차감할 수 있다.")
    @Test
    void deductQuantity() {
        // given
        Stock stock = Stock.create("001", 1);
        int quantity = 1;

        // when
        stock.deductQuantity(quantity);

        // then
        assertThat(stock.getQuantity()).isZero();
    }

    @DisplayName("재고보다 많은 수의 수량으로 차감 시도하는 경우 예외가 발생한다.")
    @Test
    void deductQuantity2() {
        // given
        Stock stock = Stock.create("001", 1);
        int quantity = 2;

        // when // then
        assertThatThrownBy(() -> stock.deductQuantity(quantity))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("차감할 재고 수량이 없습니다.");
    }
```

위와 같은 2개의 테스트를

```java
		@DisplayName("재고 차감 시나리오")
    @TestFactory
    Collection<DynamicTest> stockDeductionDynamicTest() {
        // given
        Stock stock = Stock.create("001", 1);

        return List.of(
            DynamicTest.dynamicTest("재고를 주어진 개수만큼 차감할 수 있다.", () -> {
                // given
                int quantity = 1;

                // when
                stock.deductQuantity(quantity);

                // then
                assertThat(stock.getQuantity()).isZero();
            }),
            DynamicTest.dynamicTest("재고보다 많은 수의 수량으로 차감 시도하는 경우 예외가 발생한다.", () -> {
                // given
                int quantity = 1;

                // when // then
                assertThatThrownBy(() -> stock.deductQuantity(quantity))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessage("차감할 재고 수량이 없습니다.");
            })
        );
    }
```

이렇게 변경할 수 있다!

### 테스트에서만 사용하는 메서드는..

- 만들어도 되지만, 최대한 보수적으로 만들자.
- 예를들어 DTO 를 생성하는 builder 는 생성자에서만 필요하지만, 테스트를 위해 충분히 만들 가치가 있다.

### etc..

- 구글의 구아바
    - Lists.partition() : 리스트를 여러 리스트로 쪼개서 그 쪼갠 리스트들을 리스트에 넣어서 리턴
    - Multimap  : 하나의 키는 여러 value 를 가질 수 있음.

