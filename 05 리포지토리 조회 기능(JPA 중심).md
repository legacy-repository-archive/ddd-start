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






  








