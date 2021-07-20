리포지토리 조회 기능(JPA 중심)  
==============================     
# 검색을 위한 스펙  
**리포지터리는 애그리거트의 저장소이다.**          
애그리거트를 저장하고 찾고 삭제하는 것이 리포지터리의 기본 기능이다.        
       
```java
public interface OrderRepository {
    Order findById(OrderNo id);
    List<Order> findByOrderer(String orderId, Date fromDate, Date toDate);
    ...
}
```    
검색하는 과정도 다양한 검색조건을 이용하여 다양하게 생성할 수 있다.           
이때, 네이밍 규칙으로 `findBy+개채(필드명)`을 지정하면 `DataJPA`에서 자동으로 쿼리를 만들기도한다.           
     
그런데 이렇게 검색 조건의 조합이 다양할 경우 **스펙**을 이용해서 문제를 풀어야한다.           
**스펙**은 애그리거트가 **특정 조건을 충족하는지 여부를 검사한다.**    

```java
public interface Specification<T> {
    public boolean isSatisfiedBy(T agg);
}
```  
`spring.data.jpa`에서는 이러한 스펙 검증을 위한 `Specification`인터페이스를 제공한다.   
조건 대상이 되는 애그리거트 객체를 파라미터에 넣어 `isSatisfiedBy()`에서 검증하여 true/false를 리턴한다.       
    
```java
public class OrdererSpec implements Specification<Order> {
    private String ordererId;
    public OrderSpec(String ordererId) {
        this.ordererId = ordererId;
    }
    public boolean isSatisfiedBy(Order agg) {
        return agg.getOrdererId().getMemberId().getId().equals(ordererId);
    }
}
```
**리포지터리는 스펙을 전달받아 애그리거트를 걸러내는 용도로 사용**한다.   
만약 리포지터리가 메모리에 모든 애그리거트를 보관하고 있다면 다음과 같은 스팩을 사용할 수 있다.   

```java
pubilc class MemoryOrderRepository implements OrderRepository {
    public List<Order> findAll(Sepcification sepc) {
        List<Order> allOrders = findAll();
        return allOrders.stream()
            .filter(order -> spec.isStaisfiedBy(order)).collect(toList());
    }
}
```
`Specification`는 `SimpleJpaRepository`에 한정되어    
`특정 조건`을 구현하여 이를 주입해주면 조건에 맞추어 쿼리가 실행되고 데이터를 반환한다.       
특정 조건을 충족하는 애그리거트를 찾으려면 이제 원하는 스펙을 생성해서 리포지토리에 전달해주기만 하면 된다.     

```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
List<Order> orders = orderRepository.findAll(ordererSpec);
```

## 스펙 조합 
**스펙의 장점은 `조합`이다.**    
여러 스펙을 조합해서 새로운 조건을 만들어 낼 수 있다.  

```java
public class AndSpec<T> implements Specification<T> {
    private List<Specification<T>> specs;
    
    public AndSpecification(Specification<T> ... sepcs) {
        this.specs = Arrays.asList(specs);
    }
    
    public boolean isSatisfiedBy(T agg) {
        for(Specification<t> spec : specs) {
            if(!spec.isSatisfiedBy(agg)) return false;
        }
        return true;
    }
}
```
```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
Specification<Order> orderDateSpec = new OrderDateSpec(fromDate, toDate);
AndSpec<T> spec = new AndSpec(ordererSpec, orderDateSpec);
List<Order> orders = orderRepository.findAll(spec);
```
`Specification`를 구현한 클래스를 통해 다양한 검색 조건을 스펙을 구현할 수 있다.   

# JPA를 위한 스펙 구현 
리포지터리 코드는 모든 애그리거트를 조회한 다음에 스펙을 이용해서 걸러내는 방식을 사용했다.     
하지만, 이 방식에는 실행 속도 문제가 있다.     
데이터의 갯수만큼 로딩한 다음에 다시 데이터의 갯수만큼 루프를 돌면서 검사를 진행하기 때문이다.    
      
실제 구현에서는 쿼리의 where 절에 조건을 붙여서 필요한 데이터를 걸러야 한다.        
이는 스펙 구현도 메모리에서 걸러내는 방식에서 쿼리의 where를 사용하는 방식으로 바뀌어야 한다는 것을 뜻한다.         
JPA는 다양한 검색 조건을 조합하기 위해 CriteriaBuilder와 Prdicate를 지원하므로 이를 이용해서 검색조건을 구하자  
 
## JPA 스펙 구현   
JPA를 사용하는 리포지터리를 위한 스펙의 인터페이스는 아래와 같이 정의할 수 있다.     

```java
public interface Specification<T> {
    public boolean isSatisfiedBy(T agg);
}
```  
```java
public class OrderSpec implements Specification<Order> {
    private String ordererId;
    
    public OrdererSpec(String ordererId) {
        this.ordererId = ordererId;
    }
    
    @Override
    public Prdicate toPredicate(Root<Order> root, CriteriaBuilder cb) {
        return cb.equal(root.get(Order_.orderer)
                          .get(Orderer_.memberId_.get(MemberId_.id), ordererId);
    }
}
```
`OrderSpec`의 `toPredicate()`는 Order의 `orderer.memberId.id`프로퍼티가       
생성자로 전달받은 `ordererId`와 같은지 비교하는 Predicate 를 생성해서 리턴하다.            
   
```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
List<Order> orders = orderRepository.findAll(ordererSpec);
```
응용 서비스는 원하는 스펙을 생성하고 리포지터리에 전달해서 필요한 애그리거트를 검색하면 된다.         
               
```java
public class OrdererSpecs {
    public static Specification<Order> orderer(String ordererId) {
        return (root, cb) -> cb.equals(
            root.get(Order_.orderer).get(Orderer_.memberId).get(MemberId_.id), ordererId);
    }
    
    public static Specification<Order> between(Date from, Date to) {
        return (root, cb) -> cb.between(root.get(Order_.orderDate), form, to);
    }
}
```
Specification 구현 클래스를 개별적으로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 된다.   
스펙 생성이 필요한 코드는 스펙 생성 기능을 제공하는 클래스를 이용해서 조금 더 간결하게 스펙을 생성할 수 있다. 

```java
Specification<Order> betweenSpec = OrderSpecs.between(fromTime, toTime);
```  

### JPA 정적 메타 모델 
```java
@StaticMetamodel(Order.class)
public abstract class Order_ {
    public static volatile SingularAttribute<Order, OrderNo> number;
    ...
}
```

정적 메타 모델은 `@StaticMetamodel`를 통해서 관련 모델을 지정한다.   
메타 모델 클래스의 이름은 관례적으로 맨 뒤에 `_`를 붙인다. 

정적 메타 모델 클래스는 대상 모델의 각 프로퍼티와 동일한 이름을 갖는 정적 필드를 정의한다.    
이 정적 필드는 프로퍼티에 대한 메타 모델로서 프로퍼티 타입에 따라 여러 타입을 사용해서 메타 모델을 정의한다.   

정적 메타 모델을 사용하는 대신 문자열로 프로퍼티를 지정할 수도 있다.   

```java
root.get("orderer").get("memberId").get("id")
```
하지만 문자열은 오타 가능성이 있고 실행하기 전까지 오타가 있다는 것을 놓치기 싫다.    
이런 이유로 Criteria를 사용할 때에는    
정적 메타 모델 클래스를 사용하는 것이 코드 안정성이나 생산성 측면에서 유리하다.       
   
적적 메타 모델 클래스를 직접 작성할 수 있지만 하이버네이트와 같은 JPA 프로바이더는       
정적 메타 모델을 생성하는 도구를 제공하고 있으므로 이들 도구를 사용하면 편리하다.      
     
## AND/OR 스펙조합을 위한 구현          
JPA를 위한 AND/OR 스펙은 다음과 같이 구현할 수 있다.   

```java
public class AndSpecification<T> implements Specification<T> {
    private List<Specification<T>> specs;
    
    public AndSpecification(Specification<T> ... specs) {
        this.specs = Arrays.asList(specs);
    }
    
    @Override
    public Predicate toPredicate(Root<T> root, CriterialBuilder cb) {
        Predicated[] predicates = specs.stream()
            .map(spec -> spec.toPredicate(root, cb))
            .toArray(size -> new Predicate[size]);
            return cb.and(predicates);
    }
}
```
```java
... OR 스펙은 생략 
```
`toPredicate()`는 생성자로 전달받은 `Specification` 목록으로 바꾸고       
`CriteriaBuilder`의 `and()`를 사용해서 새로운 Predicate를 생성한다.       
     
두 스펙을 쉽게 생성하기 위해 아래와 같은 클래스를 구현할 수 있다.       

```java
public class Specs {
    public static<T> Specification<T> and(Specification<T> ... specs) {
        return new AndSpecification<>(specs);
    }
    
    pubilc static<T> Specification<T> or(Specification<T> ... specs) {
        return new OrSpecification<>(specs);
    }
}
```
이제 스펙 조합이 필요하면 다음과 같은 방법으로 스펙을 조합하면 된다.   

```java
Specification<Order> specs = Specs.and(
    OrderSpecs.orderer("madvirus"), OrderSpecs.between(fromTime, toTime));
```

## 스펙을 사용하는 JPA 리포지터리 구현    
이제 남은 작업은 스펙을 사용하도록 리포지터리를 구현하는 것이다.     
먼저 리포지터리 인터페이스는 스펙을 사용하는 메서드를 제공해야한다.    

```java
public interface OrderRepository {
    public LIst<Order> findAll(Specification<Order> spec);
    ...
}
```
  
이를 상속받은 JPA 리포지토리를 같이 구현할 수 있다.    

```java
@Repository  
public class JpaOrderRepository implments OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<Order> findAll(Specification<Order> spec) {
       CriteriaBuilder cb = entityManager.getCriteriaBuilder();
       CriteriaQuery<Order> criteriaQuery = cb.createQuery(Order.class);  
       Root<Order> root = criteriaQuery.from(Order.class);
       Predicate predicate = spec.toPredicate(root, cb);
       criteriaQuery.where(predicate);
       criteriaQeury.orderBy(
           cb.desc(root.get(Order_.number).get(OrderNo_.number)));
       TypedQuery<Order> query = entityManager.createQuery(criteriaQuery);
       return query.getResultList();
    }
}
```
       
**리포지토리 구현 기술 의존**        
도메인 모델은 구현 기술에 의존하지 않아야 한다.               
그런데, JPA용 Specification 인터페이스는 toPredicate() 메서드가 JPA의 Root와            
`CirteriaBuilder`에 의존하고 있으므로 사용하는 리포지터리 인터페이스는 이미 JPA에 의존하는 모양이된다.         
   
그렇다면 `Specification`을 구현 기술에 완전히 중립적인 형태로 구형해서    













  








