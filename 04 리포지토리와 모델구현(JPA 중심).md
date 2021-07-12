리포지토리와 모델 구현(JPA 중심)     
==================================

# JPA를 이용한 리포지토리 구현     
## 모듈 위치  
`리포지터리 인터페이스`는 도메인과 같이 **도메인 영역**에 속하고           
`리포지터리 구현체`는 **인프라 스트럭처 영역**에 속한다.               
     
[#사진]()   
    
팀 표준에 따라 리포지토리 구현 클래스를 `domain.impl`과 같은 패키지에 위치시키지만 이는 좋은 설계원칙이 아니다.          
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
삭제 플래그를 사용해서 데이터를 화면에 보여줄지 여부를 결정하는 방식으로 구현한다.   
  
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
아이디가 아닌 조건으로 검색을 하고자 한다면 명명 규칙을 이용해서 메서드를 정의하면 좋다.      
이 과정에서 `JPQL`, `Criteria`, `QueryDSL` 등을 이용해도 좋으나           
`SpringDataJPA`를 활용하면 추상 메서드만으로 손쉽게 사용할 수 있다.          
     
그리고, **꼭 한개만 반환해야 된다는 것은 없기에 여러개의 엔티티(애그리거트 루트)를 반환하고자 한다면 컬렉션을 이용해도 된다.**        
  
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
그러나, **JPA는 리플랙션을 통한 객체 생성이므로 기본 생성자(파라미터 없는)가 무조건 필요하다.**        
이를 지키기 위해 무분별한 `기본 public 생성자`를 만드는 것은 객체의 안정성을 깨뜨릴 위험이 있다.   

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
**protected** 로 접근 제어자를 설정하는 것은 매우 좋은 선택이다.         
    
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









































    





















