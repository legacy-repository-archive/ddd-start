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
자바에서는 이러한 스펙 검증을 위한 `Specification`인터페이스를 제공한다.   
조건 대상이 되는 애그리거트 객체를 파라미터에 넣어      
`isSatisfiedBy()`메서드 내부에서 검증하여 true/false를 리턴한다.      
   
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
        return allOrders.stream().filter
    }
}
```





  








