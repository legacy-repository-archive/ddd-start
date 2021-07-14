리포지토리와 모델 구현(JPA 중심)     
==================================

# JPA를 이용한 리포지토리 구현     
## 모듈 위치  
`리포지터리 인터페이스`는 도메인과 같이 **도메인 영역**에 속하고           
`리포지터리 구현체`는 **인프라 스트럭처 영역**에 속한다.               
     
[#사진]()   
    
팀 표준에 따라 리포지토리 구현 클래스를 `domain` 패키지에 위치시키지만 이는 좋은 설계원칙이 아니다.          
가능하면 **리포지터리의 구현 클래스를 인프라스트럭처 영역에 위치시켜서 인프라스트럭처에 대한 의존을 낮춰야한다.**       

## 리포지터리 기본 기능 구현   
리포지터리의 기본 기능은 다음 2가지이다.     
      
* 아이디로 애그리거트 조회  
* 애그리거트 저장하기   

위 2가지를 통해 `수정/삭제`와 같은 작업도 할 수 있는 것이다.   
   
```java
public interface OrderRepository {
    public Order findById(OrderNo no);
    public void save(Order order);
}
``` 
인터페이스는 애그리거트 루트를 기준으로 작성한다.      
`주문` 애그리거트에도 여러 객체들이 존재하지만,         
`Order` 애그리거트 루트 엔티티를 기준으로 리포지터리 인터페이스를 작성한다.           

### findById
`findById()`는 특정 조건을 통해서 애그리거트 루트를 반환하는 메서드이다.      
`findById()`는 `Id`를 통해 애그리거트 루트인 `Order`를 찾거나 `null`를 리턴한다.   
더 나은 방법으로는 `Option<Order>`를 리턴하는 방식을 주로 사용한다.     

```java
    public Order findById(OrderNo id) {
        return entityManager.find(Order.class, id);
    }
```

조회 기능의 이름을 짓는데 특별한 규칙은 없지만 대부분 `findBy + 프로퍼티` 형식을 사용한다.       
더불어 `SpringDataJPA`에서는 `findBy + 프로퍼티` 명명 규칙에 따른 쿼리 메서드를 제공하기도 한다.        
    
### save   
파라미터 값으로 `애그리거트 루트`를 원하며 **파라미터로 전달받은 애그리거트를 저장한다.**       
JPA를 공부했으면 들어 봤을 `EntityManager`를 이용해서 기능을 구현한다.     

```java
    public Order save(Order order) {
        return entityManager.save(order);
    }
```

### delete
애그리거트를 삭제하는 기능이 필요할 때도 있다.   

```java
    public void delete(Order order) {
        entityManager.remove(order);
    }
```
애그리거트를 객체를 파라미터로 받아 `entityManager.remove()`에 주입하면 삭제가 진행된다.   

**삭제 기능**   
삭제 요구사항이 있더라도 여러 이유로 데이터를 실제로 삭제하는 경우는 많지 않다.         
관리자 기능에서 삭제한 데이터까지 조회해야하는 경우도 있고          
데이터 원복을 위해 일정 기간 동안 보관해야 할 때도 있다.         
    
이런 이유로 사용자가 삭제 기능을 실행할 때 바로 삭제하기 보다는         
**삭제 플래그를 사용해서 데이터를 화면에 보여줄지 여부를 결정하는 방식으로 구현한다.**      
  
### 예시   

```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Order findById(OrderNo id) {
        return entityManager.find(Order.class, id);
    }
    
    @Override
    public Order save(Order order) {
        return entityManager.save(order);
    }
    
    @Override
    public Order delete(Order order) {
        return entityManager.remove(order);
    }
    
}
```
     
참고로, 엔티티로 사용되는 애그리거트 루트는 `더티체킹`으로 인해 값을 수정하면 DB도 반영된다.     
(트랜잭션 범위내로 한정되어있다. 즉, 트랜잭션이 끝나면 변경된 데이터를 DB에 반영한다.)        

```java
public interface OrderReopository {
    ...
    public List<Order> findByOrdererId(String ordererId, int startRow, int size);
}
```    
아이디가 아닌 조건으로 검색을 하고자 한다면 **명명 규칙을 이용해서 메서드를 정의하면 좋다.**         
이 과정에서 `JPQL`, `Criteria`, `QueryDSL` 등을 이용해도 좋으나           
`SpringDataJPA`를 활용하면 추상 메서드만으로 손쉽게 사용할 수 있다.            
그리고, **여러개의 엔티티(애그리거트 루트)를 반환하고자 한다면 컬렉션을 이용해도 된다.**        
  
# 매핑 구현  
애그리거트와 JPA 매핑을 위한 기본 규칙은 아래와 같다.  
 
* 애그리거트 루트는 엔티티므로 `@Entity` 로 매핑 설계한다.      
* 한 테이블에 엔티티와 벨류 데이터가 같이 있다면       
  * 벨류 클래스 : `@Embeddable`로 매핑 설정한다.     
  * 엔티티에 존재햐는 벨류 타입 프로퍼티는 `@Embedded`로 매핑설정한다.     

루트 엔티티와 루트 엔티티에 속한 벨류는 DB의 한 테이블에 매핑될 때가 많다.   

[#설계]() 

## 일반적인 @Embeddable 과 @Embedded 사용 

```java
@Entity
@Table(name = "purchase_order")
public class Order {
   ... 
}
```
```java
@Embeddable
public class Orderer {
    @Embedded
    @AttributeOverrides(
        @AttributeOverride(name="id", column = @Column(name="orderer_id"))
    )
    private MemberId memberId;
    
    ...
}
```
```java
@Embeddable  
public class MemberId implements Serializalbe {
    @Column(name="member_id")
    private String id;
}
```
설계에 따라 위와 같이 코드를 작성할 수 있다.     
큰 특징은 아니지만, 벨류와 설계 이름이 다르므로 `@AttributeOverride`를 통해 컬럼명을 바꾸었다.    

```java
@Embeddable  
public class ShippingInfo {
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name="zipcode", column=@Column(name="shipping_zipcode")),
        @AttributeOverride(name="address1", column=@Column(name="shipping_addr1")),
        @AttributeOverride(name="address2", column=@Column(name="shipping_addr2")),
    })
    private Address address;
    
    @Embedded
    private Receiver receiver;
}
```
```java
@Entity
public class Order {
    ... // 생략 
    @Embedded
    private Orderer orderer;
    
    @Embedded
    private ShippingInfo shippingInfo;
    ... // 생략  
}
```

## 기본 생성자   
엔티티와 벨류의 생성자는 객체를 생성할 때 필요한 것을 전달 받는다.     
이와 별개로 **JPA는 리플랙션을 통한 객체 생성이므로 기본 생성자(파라미터 없는)가 무조건 필요하다.**        
그러나, 이를 지키기 위해 `기본 public 생성자`를 만드는 것은 객체의 안정성을 깨뜨릴 위험이 있다.     
  
```java
@Embeddable
public class Receiver {
    @Column(name="receiver_name")  
    private String name;
    @Column(name="receiver_phone")  
    private String phone;
    
    protected Receiver() {}
    
    public Receiver(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }
    
    ... // 생략
}
```
`JPA 프로바이더`에서 사용되어야 하기 위해 기본 생성자를 정의해야 한다면           
**protected 로 접근 제어자를 설정하는 것은 매우 좋은 선택이다.**              
        
**노트**       
하이버네이트는 클래스를 상속한  프록시 객체를 이용해서 지연로딩을 구현한다.    
이 경우 프록시 객체에서 상위 클래스의 기본 생성자를 호출할 수 있어야 하므로      
지연 대상이 되는 `@Entity` 와 `@Embeddable` 기본 생성자는 `private/default`가 아니어야 한다.      
      
## 필드 접근 방식  
JPA는 컬럼을 등록할 때 `필드/메서드` 2가지 방식을 지원한다.                   
단, 메서드 방식은 `getter/setter`를 구현해야하기 때문에 불변을 깰 수 있다.               
더불어 `changeShippingInfo()`가 아닌 의미를 찾기 힘든 `setShippingInfo`로 메서드를 정의해야한다.   

엔티티를 객체가 제공할 기능 중심으로 구현하도록 유도하려면    
JPA매핑 처리를 프로퍼티 방식이 아닌 필드 방식으로 선택해서 불필요한 getter/setter 메서드 구현을 말아야한다.   

```java
@Entity
@Access(AccessType.FILED)   
public class Order {
    ... // 정의
}
```
JPA 구현체인 하이버네이트는 `@Access`를 이용해서 명시적으로 접근 방식을 지정하지 않으면       
`@Id`나 `@EmbeddedId`가 어디있는지에 따라 접근 방식을 결정한다.        
필드에 위치하면 필드, 메서드에 위치하면 메서드 접근 방식을 지원한다.       
그래도 필드에 지정하면 알아서 되므로 굳이 작성하는 방향으로 가지는 않아도 될 것 같다.    
   
## AttributeConverter 이용한 밸류 매핑 처리  
int, long, String, LocalDate 와 같은 타입은 DB 테이블의 한 개 컬럼과 매핑된다.     
마찬가지로, **벨류 타입의 프로퍼티를 한 개의 컬럼에 매핑해야할 때도 있다.**       
즉, 벨류 내에 존재하는 2개 이상의 프로퍼티를 한 개의 컬럼으로 매핑하는 것을 말한다.     

```java
public class Product {
    @Column(name="WIDTH")
    private String width;
    
    public Length getLength() {
        return new Width(width);        // DB 컬럼 값을 실제 프로파티 타입으로 변환한다.   
    }
    
    void setWidth(Length length) {
        this.width = width.toString();  // 실제 프로퍼티 타입을 DB 컬럼 값으로 변환  
    }
}
```
`JPA 2.0` 이하에서는 위와 같이 `getter/setter`를 이용해서 값을 처리할 수 밖에 없었다.    

```java
public interface AttributeConverter<X,Y> {
    public Y convertToDatabaseColumn(X attribute);     // 단순 벨류 타입 -> DB 컬럼 변환   
    public X convertToEntityAttribute(Y dbDate);       // DB 타입 -> DB 컬럼값을 벨류로 변환   
}
```
```java
@Converter(autoApply=true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {
    
    @Override
    public Integer convertToDatabaseColumn(Money money) {
        if(money == null) { return null; }
        else return money.getValue();
    }
    
    @Override
    public Money convertToEntityAttribute(Integer value) {
        if(value == null) return null;
        else return new Money(value);
    }
}
```
`AttributeConverter`는 `엔티티<->벨류`를 지원하는 인터페이스로 `<>`에는 컨버트할 타입들을 적어놓는다.       
`AttributeConverter<>`를 구현한 클래스는 `@Converter()`을 사용한다.   
`autoApply=true`로 함으로써 모델에 출현하는 **모든 `Money` 타입의 프로퍼티**에 대해 `MoneyConverter`를 자동으로 적용한다.     

```java
public class Order {
    
    @Column(name = "total_amounts")
    @Convert(converter = MoneyConverter.class)
    private Money totalAmounts;
    
    ...
}
```
`autoApply=false`일 경우,    
**사용하고자 하는 프로퍼티에만 `@Convert(converter = 컨버터클래스.class)`를 적어주어 적용시킨다.**   
   
## 벨류 컬렉션:별도 테이블 매핑  
JPA `1 대 N 관계`를 표현하면 아래와 같다.   

```java
public class Order {
    private List<OrderLine> orderLines;
}
```
**벨류 타입의 컬렉션은 별도 테이블에 보관한다.**  

[#사진]()  

벨류 컬렉션과 매핑된 `ORDER_LINE` 테이블은     
외부키를 이용해서 엔티티에 해당하는 `PURCHASE_ORDER`테이블을 참조한다.     
이 외부키는 컬렉션이 속할 엔티티를 의미한다.      
`List 타입 컬렉션`은 인덱스 값이 필요하므로 `ORDER_LINE` 테이블에는 인덱스 값을 저장하기 위한 칼럼도 존재한다.        
       
```java
public class Order {
     ...
     @ElementCollection
     @CollectionTable(name="order_line", joinColumns=@JoinColumn(name="order_number"))
     @OrderColumn(name="line_idx")
     private List<OrderLine> orderLines;
     ...
}
```
```java
@Embeddable
public class OrderLine {
    @Embedded
    private ProductId productId;
    
    @Column(name="price")
    private Money price;
    
    @Column(name="quantity")
    private int quantity;
    
    @Column(name="amounts")
    private Money amounts;
    
    ...
}
```
`OrderLine`의 매핑을 함께 표시했는데         
`OrderLine`에는 `List`의 인덱스 값을 저장하기 위한 프로퍼티가 존재하지 않는다.              
그 이유는 List 타입 자체가 인덱스를 갖고 있기 때문이다.      
JPA는 `@OrderColumn`을 이용해서 지정한 컬럼에 리스트 인덱스 값을 저장한다.     
   
`@CollectionTable`은 벨류를 저장할 테이블을 지정할 때 사용한다.             
`name` 속성으로 테이블 이름을 지정하고 `joinColumns` 속성은 외부키로 사용하는 컬럼을 지정한다.        
외부키가 2개 이상인 경우애는 `@JoinColumn`의 배열을 이용해서 외부키 목록을 지정한다.     
  
## 벨류 컬렉션: 한 개 컬럼 매핑     
**벨류 컬렉션**을 별도 테이블이 아닌 한 개 컬럼에 저장해야 될 때가 있다.        
예를들어, 이메일을 `Set<>`으로 보관하고 DB에는 한 개 컬럼에 콤마로 구분해서 저장해야할 때가 있다.         
이때, `AttributeConverter`를 이용하면 벨류 컬렉션을 한 개 컬럼에 쉽게 매핑할 수 있다.      
단, 이를 이용하려면 **컬렉션을 표현하는 새로운 벨류 타입 클래스를 추가해야한다.**        

```java
public class EmailSet {
    private Set<Email> emails = new HashSet<>();
    
    private EmailSet(){}
    privat EmailSet(Set<Email> emails) {
        this.emails.addAll(emails);
    }
    
    public Set<Email> getEmails() {
        return Collections.unmodifiableSet(emails); 
    }
}
```
```java
@Converter
public class EmailSetConverter implements AttributeConverter<EmailSet, String> {
    @Override
    public String convertToDatabaseColumn(EmailSet attribute) {
        if(attribute == null) return null;
        return attribute.getEmails().stream()
            .map(Email::toString())
            .collect(Collectors.joining(",");
    }
    
    @Override
    public String convertToEntityAttributes(String dbData) {
        if(dbData == null) return null;
        String[] emails = dbData.split(",");
        Set<Email> emailSet = Arrays.stream(emails)
            .map(value -> new Email(value))
            .collect(toSet());
        return new EmailSet(emailSet);
    }
}
```
```java
@Column(name="emails")
@Convert(converter = EmailSetConverter.class)   
private EmailSet emailSet;
```
   
## 벨류를 이용한 아이디 매핑      
식별자는 최종적으로 문자열이나 숫자와 같은 기본 타입이기에 `String` 이나 `Long` 타입을 이용해서 식별자를 매핑한다.      

```java
@Entity
public class Order {
    // 기본 타입을 이용한 식별자 매핑  
    @Id
    private String number;
    ...
}
```
```java
@Entity
public class Article {
    @Id
    private Long id;
    ...
}
```   
기본 타입을 사용하는 것도 좋지만, 벨류 타입 객체로 만들어 사용하는 것도 좋은 방법이다.    
식별자를 벨류타입으로 매핑한다면 `@EmbeddedId`를 사용해야한다.   
   
```java
@Entity
@Table(name = "purchase_order")
public class Order {
    // 기본 타입을 이용한 식별자 매핑  
    @EmbeddedId
    private OrderNo number;
    ...
}

@Embeddedable
public class OrderNo implements Serializable {
    @Column(name="order_number")
    private String number;
    ...
}
```
JPA에서 식별자 타입은 `Serializable` 이어야 하므로 인터페이스 구현을 해줘야한다.     

벨류 타입 식별자의 정점은 식별자에 기능을 추가할 수 있다는 점이다.    
예) `1세대 주문 번호`와 `2세대 주문번호` 구분시 첫 글자를 이용한다.     

```java
@Embeddedable
public class OrderNo implements Serializable {
    @Column(name="order_number")
    private String number;
    
    public boolean is2ndGeneration() {
        return number.startWith("N");  
    }
}
```
```java
if (order.getNumber().is2ndGeneration()) {
   ...
}
```
시스템 세대 구분이 필요한 코드는 OrderNo가 제공하는 기능을 이용해서 구분하면된다.     
JPA는 내부적으로 엔티티를 비교할 목적으로 `equals()/hashcode()`를 통해 속한 모든 필드를 비교해야한다.   
  
## 별도 테이블에 저장하는 벨류 매핑   
애그리거트에서 루트 엔티티를 제외한 나머지 구성요소는 대부분 벨류이다.       
그렇기에 **또 다른 엔티티가 있다면 진짜 엔티티인지 의심해봐야 한다.**      
예를 들면, `OrderLine`은 별도의 테이블을 가지고 있지만 엔티티가 아닌 벨류이다.        
    
**벨류가 아닌 엔티티가 확실하다면 다른 애그리거트는 아닌지 확인해야한다.**              
특히, 자신만의 독자적인 라이프사이클을 갖는다면 다른 애그리거트일 가능성이 높다.         
앞선 챕터에서 언급했듯이 상품과 상품 리뷰는 독자적인 라이프사이클을 가지기에 다른 애그리거트이다.     
        
애그리거트에 속한 객체가 벨류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지 여부를 확인하는 것이다.         
하지만, 식별자를 찾을 때 **매핑되는 테이블의 식별자를 애그리거트 구성요소의 식별자와 동일한 것으로 착각하면 안 된다.**          
**별도 테이블로 저장되고 테이블에 PK가 있다고 해서 테이블과 매핑되는 애그리거트 구성요소가 고유 식별자를 갖는 것은 아니다.**   

예를 들어, 게시글 데이터를 ARTICLE 테이블과 ARTICLE_CONTENT 테이블로 나눠서 저장한다고 가정한다.     

[#사진](#)     
     
ARTICLE_CONTENT 테이블의 ID 컬럼이 식별자이므로       
ARTICLE_CONTENT와 매핑되는 ArticleContent를 엔티티로 생각할 수 있기에     
Article과 ArticleContent 두 엔티티 간의 일대일 연관으로 매핑하는 실수를 할 수 있다.   
   
ArticleContent는 Article의 벨류다.     
ARTICLE_CONTENT의 ID는 식별자이기는 하지만         
ARTTICLE 테이블과 연결하기 위함일뿐 글의 특정 프로퍼티를 별도 테이블에 보관한 것으로 접근해야한다.          
(앞서 ID 기법에 대해서 말했는데 이를 적용한 것 같다.)      
  
ArticleContent는 벨류이므로 `@Embeddable`로 매핑한다.      
ArticleContent와 매핑되는 테이블은 Article과 매핑되는 테이블과 다른데,   
이때 벨류를 매핑한 테이블을 지정하기 위해 `@SecondaryTable`과 `@AttributeOverride`를 사용한다.   

```java
@Entity
@Table(name="article")   
@SecondaryTable(
    name="article_content",
    pkJoinColumns=@PrimaryKeyJoinColumn(name="id")
)
public class Article {
    @Id
    private Long id;
    private String title;
    ...
    
    @AttributeOverrides({
        @AttributeOverride(name="content", column=@Column(table="artilce_content"}),
        @AttributeOverride(name="contentType", column=@Column(table="artilce_content"})
    })
    private ArticleContent content;
    ...
}
```
`@SecondaryTable`의    
`name 속성`은 벨류를 지정할 테이블을 지정한다.        
`pkJoinColumns 속성`은 벨류 테이블에서 엔티티 테이블로 조인할 때 사용할 컬럼을 지정한다.        
content 필드에 `@AttributeOverride`를 사용해서 새로운 컬럼 값을 정의해줬다.   
     
```java   
Article article = entityManager.find(Article.class, 1L);    
```
`@SecondaryTable`을 이용하면 아래 코드를 실행할 때 두 테이블을 조인해서 데이터를 조회한다.          

* Article
* ArticleCotent  

게시글은 ARTICLE 테이블만 필요하지 ARTICLE_CONTENT는 필요하지 않는다.     
`@SecondTable`은 벨류인 테이블을 조인해서 값을 가져오는데 **이는 원하는 결과가 아니다.**  

ArticleContent 를 별도의 엔티티로 만들어 즉시/지연로딩을 할 수 있지만     
이 방식은 엔티티가 아닌 모델을 엔티티로 만드는 것으로 좋은 방법은 아니다.     
대신 조회 전용 기능을 구현하는 방법을 사용할 수 있는데 5장에서 다뤄본다.    
    
## 벨류 컬렉션을 @Entity로 매핑하기       
개념적으로 **벨류인데 구현 기술의 한계나 팀 표준으로 @Entity를 사용해야 할 때도 있다**         
제품의 이미지 업로드 방식에 따라 이미지 경로와 썸네일 이미지 제공 여부가 달라진다고 가정하자        
   
[#](#)     
   
JPA는 `@Embeddable`타입의 클래스가 상위 클래스가 되는 상속 매핑을 지원하지 않는다.            
따라서 상속 구조를 갖는 밸류 타입을 사용하려면 `@Embeddable` 대신 `@Entity`를 이용한 상속 매핑으로 처리한다.                   
또한, 구현 클래스를 기본하기 위한 타입 식별 컬럼을 추가해야한다.           
     
[#](#)
   
한 테이블에 Image 및 하위 클래스를 매핑하므로      
`Image 클래스`에 `@Inheritance`를 적용하고 strategy 값으로 `SINGLE_TABLE`을 사용하고,           
`@DiscriminatorColumn`을 이용해서 타입을 구분하는 용도로 사용할 컬럼을 지정한다.              
`Image`를 `@Entity`로 매핑했지만 모델에서 `Image는 엔티티`가 아니라 벨류이므로                 
아래 코드와 같이 상태를 변경하는 기능은 추가하지 않는다.        
   
```java
@Entity
@Inheritance(Strategy=InheritanceYpe.SINGLE_TABLE)
@DiscriminationColumn(name="image_type")   
@Table(name="image")
public abstract class Image {
    ...
    public abstract String getUrl();
    public abstract boolean hasThumbnail();
    public abstract String getThumbnailURL();
}
```
`Image`를 상속받은 클래스는 다음과 같이 `@Entity`와 `@Discriminitor`를 사용해서 매핑을 설정한다.     
     
```java
@Entity
@DiscriminatorValue("II")
public class InternalImage extends Image {
    ...
}
```
```java
@Entity
@DiscriminatorValue("EI")
public class ExternalImage extends Image {

}
```

`Image`가 `@Entity`이므로    
`Image 목록`을 담고 있는 Product는 `@OneToMany`를 이용해서 매핑을 처리한다.             
Image는 벨류이므로 독자적인 라이프사이클을 갖지 않고 Product에 완전히 의존한다.             
따라서 cascade 속성을 이용해서 Product를 저장할 때 함께 저장되고, 
Product를 삭제할 때 함께 삭제되도록 설정한다.           
리스트 Image 객체를 제거하면 DB에서 함께 삭제되도록 orphanRemoval도 true로 설정한다.       

```java
@Entity
@Table(name="product")
public class Product {
    @EmbeddedId
    private ProductId id;
    private String name;
    
    @Convert(converter=MoneyConverter.class)
    private Money price;
    private String detail;
    
    @OneToMany(cascade={CascadeType.PERSIST, CascadeType.REMOVE}, orphanRemoval=true) 
    @JoinColumn(name="product_id")
    @OrderColumn(name="list_idx")
    private List<Image> images = new ArrayList<>();
    ...
    public void changeImages(List<Image> newImages) {
        images.clear();
        images.addAll(newImages);
    }    
}
```
`changeImages()`는 이미지 교체를 위해 `List<>`의 `clear()`를 사용한다.    
하지만, `OneToMany` 구조에서 `clear()`는 각각의 대상을 쿼리로 조회하고 삭제한다.        
즉, 전체를 가져오는 1번, 각각의 객체를 삭제하는`N`이되어 `N+1`이라는 무시무시한 쿼리를 날리게 된다.        
  
하이버네이트는 `@Embeddable`에 대한 컬렉션의 `clear()`를 호출하면      
컬렉션에 속한 객체를 로딩하지 않고 한 번의 `delete`로 삭제 처리를 수행한다.     
   
애그리거트의 특성을 유지하면서 이 문제를 해소하려면 결국 상속을 포기하고   
`@Embeddable`로 매핑된 단일 클래스로 구성해야한다.     
물론 이 같은 타입에 따라 기능을 구현하려면 다음과 같이 if-else를 써야하는 단점이 있다.   

  
```java
@Embeddable
public class Image {
    @Column(name="image_type")
    private String imageType;
    @Column(name="iamge_path")
    private String path;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name="upload_time")
    private Date uploadTime;
    ...
    
    public boolean hasThumbnail() {
        if (imageType.equals("II")) {
            return true;
        } else {
            return false;
        }
    }
}
```
코드 유지보수와 성능의 두 가지 측면을 고려해서 구현 방식을 선택해야한다.   
  
## ID 참조와 조인 테이블을 이용한 단방향 M:N 매핑           
애그리거트간 집합 연관은 성능상의 이유로 피해야 한다고 했다.       
그럼에도 불구하고 요구사항을 구현하는데 집합 연관을 사용하는 것이 유리하다만          
`ID 참조`를 이용한 단방향 집합 연관을 적용해 볼 수 있다.     

```java
@Entity  
@Table(name="product")      
public class Product {
    @EmbeddedId
    private ProductId id;
    
    @ElementCollection
    @CollectionTable(name="product_category",  
        joinColumns = @JoinColumn(name="product_id")) 
    private Set<CategoryId> categoryIds;
}
```

이코드는 `Product`에서 `Category`로의 단방향 M:N 연관을 ID참조 방식으로 구현한 것이다.   
ID참조를 이용한 애그리거트 간 단방향 M:N 연관은 벨류 컬렉션 매핑과 동일한 방식으로 설정한 것을 알 수 있다.  
차이점이 있다면 집합의 값에 벨류 대신 연관을 맺는 식별자가 온다는 점이다.   
  
`@ElementCollection`을 이용하기 때문에 Product를 삭제할 때         
매핑에 사용한 조인 테이블의 데이터도 함께 삭제된다.      
애그리거트를 직접 참조하는 방식을 사용햇다면 영속성 전파나 로딩 전략을 고민해야하는데     
ID 참조 방식을 사용함으로써 이런 고민을 할 필요가 없다.   

# 애그리거트 로딩 전략   
JPA 매핑에서 가장 중요한 점은 애그리거트에 속한 객체가 모두 모여야 완전한 하나가 된다는 것이다.     

```java
Product product = productRepository.findById(id);
```
조회 시점에서 애그리거트를 완전한 상태가 되도록 하려면        
**루트 연관 매핑의 조회 방식을 즉시 로딩으로 설정하면 된다.**          

```java
// @Entity 컬렉션에 대한 즉시 로딩 설정  
@OneToMany(cascade={CascadeType.PERSIST, CascadeType.REMOVE},  
    orphanReomval=true, fetchFetchType.EAGER
@JoinColumn(name="product_id")    
@OrderColumn(name="list_idx")      
private List<Image> images = new ArrayList<>();
  
// @Embeddable 컬렉션에 대한 즉시 로딩 설정   
@ElementCollection(fetch=FetchType.EAGER)
@CollectionTable(name="order_line",
    joinColumns = @JoinColumn(name="order_number"))
@OrderColumn(name="line_idx")
private List<OrderLine> orderLines;
```
`즉시로딩`을 사용하면 로딩 시점에 연관된 객체를 한번에 로딩한다는 장점이 있지만,   
컬렉션에 대해서 로딩한다면 심각한 성능 이슈를 발생시킨다.   

```java
@Entity
@Table(name="product")
public class Product {
    ...
    @OneToMany(
        cascade={CascadeType.PERSIST, CascadeType.REMOVE},
        orphanRemoval = true,
        fetch=FetchType.REAGER)
    @JoinColumn(name="product_id")
    @OrderColumn(name="list_idx")
    private List<Image> images = new ArrayList<>();
    
    @ElementCollection(fetch=FetchType.EAGER)
    @CollectionTable(name="product_option",  
        JoinColumns=@JoinColumn(name="product_id"))
    @OrderColumn(name="list_idx")
    private List<Option> options = new ArrayList<>();
    ...    
}
```
`fetch=FetchType.EAGER`를 이용하면 값을 가져올때 조인해서 가져온다.       
단, 이 쿼리는 **카타시안 조인**을 사용하는데 쿼리 결과에 중복을 발생시킨다.          
즉, 조회하는 Product의 image가 2개이고 option이 2개면, 쿼리 결과로 구해지는 행 개수는 4개다.         
즉, Product는 4번 중복되고 image와 product_option은 2번 중복된다.     
    
물론, 하이버네이트가 중복된 데이터를 알맞게 제거해서       
실제 메모리에는 1개의 Product 객체, 2개의 Image 객체, 2개의 Option 객체로 변환해 주지만     
애그리거트가 커지면 문제가 될 수 있다.(10개/25개면 -> 250개)      
보통 조회 성능때문에 즉시로딩을 사용하지만 이경우 오히려 즉시로딩 때문에 성능이 나빠진 경우다.        

애그리거트는 개념적으로 하나여야하지만, 루트 엔티티를 로딩하는 시점에 모든 객체를 로딩할 필요는 없다.     
만약, 애그리거트가 완전해야 한다면 그 상황으로 2가지가 있다.   
      
* 상태를 변경하는 기능을 실행할 때         
* 표현 영역에서 애그리거트의 상태 정보를 보여줄 때     
        
이 중 `표현 영역에서 애그리거트의 상태 정보를 보여줄 때`는          
별도의 조회 전용 기능을 구현하는 방식을 사용하는 것이 유리할 때가 많기 때문에              
애그리거트의 완전한 로딩과 관련된 문제는 상태 변경과 더 관련이 있다.            
상태 변경 기능을 실행하기 위해 조회 시점에 즉시 로딩을 이용해서 애그리거트를 완전한 상태로 로딩할 필요는 없다.         
JPA는 트랜잭션 범위 내에서 지연 로딩을 허용하기 때문에       
다음 코드처럼 실제로 상태를 변경하는 시점에 필요한 구성요소만 로딩해도 문제가 되지 않는다.          

```java
@Transactional   
public void removeOptions(ProductId id, int optIdxToBeDeleted) {
    // Product를 로딩, 컬렉션은 지연 로딩으로 설정했다면, option은 로딩하지 않는다.   
    Product product = productRepository.findById(id); 
    // 트랜잭션 범위이므로 지연 로딩으로 설정한 연관 로딩 가능  
    product.removeOption(optIdxToBeDeleted);
}
```
```java
@Entity
public class Product {
    
    @ElementCollection(fetch=FetchType.LAZY)
    @CollectionTable(name="product_option", 
        joinColumns=@JoinColumn(name="product_id"))
    @OrderColumn(name="list_idx")
    private List<Option> options = new ArrayList<>();
    
    public void removeOption(int optIdx) {
        // 실제 컬렉션에 접근할 때 로딩  
        this.options.remove(optIdx);
    }
}
```
일반적인 애플리케이션은 상태를 변경하는 기능을 실행하는 빈도보다 조회하는 기능을 실행하는 빈도가 훨씬 높다.         
그러므로 상태 변경을 위해 지연 로딩을 사용할 때 발생하는 추가 쿼리로 인한 실행 속도 저하는 문제가 되지 않는다.     
       
이런 이유로 애그리거트 내의 모든 연관을 즉시 로딩으로 설정할 필요는 없다.         
지연 로딩은 동작 방식이 항상 동일하기 때문에 즉시 로딩처럼 경우의 수를 따질 필요가 없는 장점이 있다.          
       
즉시 로딩 설정은 `@Entity`나 `@Embeddable`에 대해 다르게 동작하고,       
JPA프로바이더에 따라 구현 방식이 다를 수 있다.      
물론, 지연로딩은 즉시 로딩모다 쿼리 실행 횟수가 많아질 가능성이 더 높다.      
따라서 무조건 즉시로딩이나 지연로딩으로만 설정하기 보다는 애그리거트에 맞게 즉시 로딩과 지연로딩을 선택해야한다.      

# 애그리거트 영속성 전파  
애그리거트가 완전한 상태여야 한다는 것은     
애그리거트 루트를 조회할 때뿐만 아니라 저장하고 삭제 할때도 하나로 처리해야 함을 의미한다.       
            
* 저장 메서드는 애그리거트 루트만 저장하면 안 되고 애그리거트에 속한 모든 객체를 저장해야 한다.         
* 삭제 메서드는 애그리거트 루트 뿐만 아니라 애그리거트에 속한 모든 객첼를 삭제해야한다.            
           
`@Embeddable` 매핑 타입의 경우 함께 저장되고 삭제되므로 `cascade` 속성을 추가로 설정하지 않아도 된다.            
반면에 애그리거트에 속한 `@Entity`타입에 대한 매핑은 `cascade`속성을 사용해서 저장과 삭제 시에 함께 처리되도록 설정해야한다.         
`@OneToOne`, `@OneToMany` 는 cascade 속성의 기본값이 없으므로        
다음 코드처럼 cascade 속성값으로 CascadeType.PERSIST, CascadeType.REMOVE 를 설정한다.     
   
```java
@OneToMany(cascade={CascadeType.PERSIST, CascadeType.REMOVE},
    orphanRemoval = true)
@JoinColumn(name="product_id")   
@OrderColumn(name="list_idx")
private List<Image> iamges = new ArrayList<>();
```

# 식별자 생성 기능  
식별자는 크게 3가지 방식 중 하나로 생성한다.   
    
* 사용자가 직접 생성          
* 도메인 로직으로 생성         
* DB를 이용한 일련번호 사용        

이메일 주소처럼 사용자가 직접 식별자를 입력하는 경우는 식별자 생성 주체가 사용자이기 때문에       
도메인 영역에 식별자 생성 기능을 구현할 필요가 없다.     
       
식별자 생성 규칙이 있는 경우 엔티티를 생성할 때 이미 생성한 식별자를 전달하므로         
엔티티가 식별자 생성 기능을 제공하는 것 보다는 별도 서비스로 식별자 생성 기능을 분리해야한다.      
식별자 생성 규칙은 도메인 규칙이므로 도메인 영역에 식별자 생성 기능을 위치시켜야한다.     
예를 들어 다음과 같은 도메인 서비스를 도메인 영역에 위치시킬 수 있다.   
  
```java
public class ProductIdService {
    public ProductId nextId() {
        ... // 정해진 규칙으로 식별자 생성  
    }
}
```
응용 서비스는 이 도메인 서비스를 이용해서 식별자를 구하고 엔티티를 생성할 것이다.   

```java
public class CreateProductService {
    @Autowired private ProductIdService idService;
    @Autowired private ProductRepository productRepository;
      
    @Transactional   
    public ProductId createProduct(ProductCreationCommand cmd) {
        // 응용 서비스는 도메인 서비스를 이용해서 식별자를 생성   
        ProductId id = productIdService.nextId();   
        Product product = new Product(id, cmd.getDetail(), cmd.getPrice(), ...);
        productRepository.save(product);
        return id;
    }
}
```
특정 값의 조합으로 식별자가 생성되는 것 역시 규칙이므로 도메인 서비스를 이용해서 식별자를 생성할 수 있다.       
예를 들어, 주문번호를 고객 ID와 타임 타임스탬프로 구성한다고 할 경우 다음과 같은 도메인서비스를 구현할 수 있다.    

```java
public class OrderIdService {
    public OrderId createId(UserId userId) {
        if(userId == null) {
            throw new IllegalArgumentException("invalid userid: " + userId);   
        }  
    }
    
    private String timestamp() {
        return Long.toString(System.currentTimeMillis());
    }
}
```

식별자 생성 규칙을 구현하기에 적합한 또 다른 장소는 리포지터리이다.     
다음과 같이 리포지터리 인터페이스에 식별자를 생성하는 메서드를 추가하고 리포지터리 구현 클래스에서 알맞게 구현하면 된다.     
   
```java   
public interface ProductRepository {
    ...// save() 등 다른 메서드 
    
    // 식별자를 생성하는 메서드  
    ProductId nextId();
}
```

식별자 생성으로 DB의 자동 증가 컬럼을 사용할 경우 JPA의 식별자 매핑에서 `@GeneratedValue`를 사용한다.   

```java
@Entity 
@table(name="article")     
...
public class Article {
    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)   
    private Long id;
    
    public Long getId() {
        return id;
    }
}
```
자동 증가 컬럼은 DB의 insert 쿼리를 실행해야 식별자가 생성되므로 도메인 객체를 리포지터리에 저장할 때 식별자가 생성된다.     
이 이야기는 도메인 객체를 생성하는 시점에는 식별자를 알 수 없고 도메인 객체를 저장한 뒤에 식별자를 구할 수 있음을 의미한다.     

```java
public class WriteArticleService {
   private ArticleRepository articleRepository;   
   
   public Long write(NewArticleRequest req) {
       Article article = new Article("제목", new ArticleContent("content", "type"));
       articleRepository.save(article); // EntityManger#save()
                                        // 실행 시점에 식별자 생성
       return article.getId();          // 저장 이후 식별자 사용 가능                                   
   }
   ...
}
```
JPA는 저장 시점에 생성한 식별자를 `@Id`로 매핑한 프로퍼티/필드에 할당한다.        
따라서 위 코드처럼 저장 이후에 엔티티의 식별자를 사용할 수 있다.             
      
자동 증가 컬럼외에 JPA의 식별자 생성 기능을 사용하는 경우에도 마찬가지로 저장시점에 식별자를 생성한다.         
