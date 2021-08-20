도메인 모델과 BoundedContext     
==============================    
# 도메인 모델과 경계     
도메인 모델 설계시 흔히 하는 실수가 **도메인을 완벽하게 표현하는 단일 모델을 만드는 것이다.**     
앞서 서적 초반에 언급했듯이 도메인은 그 하위 도메인으로 구성되어있기에      
한 개의 모델로 여러 하위 도메인을 모두 표현하려고하면 모든 하위 도메인에 맞지 않는 모델을 만들게 된다.      
   
**이름은 같지만 다른 역할**   
* **카탈로그 상품 :** 상품 이미지, 상품명, 상품 가격과 같은 **상품 정보**  
* **재고관리 상품 :** 실존하는 개별 객체를  추적하기 위한 목적   

이들은 이름만 같지 실제로 의미하는 것은 다르다.       
카탈로그에서 물리적으로 1개인 상품은, 재고관리에서 여러 개 존재할 수 있다.        
즉, **논리적으로 같은 존재처럼 보이지만 `하위 도메인`에 따라 다른 용어를 사용하는 경우도 있다.**     
    
[]()     
 
**이름은 다르지만 같은 역할**   
* **회원 도메인 :** 회원
* **주문 도메인 :** 주문자
* **배송 도메인 :** 보내는 사람
      
반대로 같은 대상이라도 지칭하는 용어가 다를 수 있다.          
**한 개의 모델로 모든 하위 도메인을 표현하려는 시도는 올바른 방법이 아니며 표현할 수도 없다.**     
     
```java
// User 클래스가 모든 도메인에서의 사용자를 대표할 수 없다는 뜻이다.      
public class User {
}
```     
  
하위 도메인마다 사용하는 용어가 다르기 때문에       
**올바른 도메인 모델을 개발하려면 `하위 도메인마다` 모델을 만들어야 한다.**         

* **회원 도메인 :** Member.class
* **주문 도메인 :** Orderer.class
* **배송 도메인 :** Sender.class
     
각 모델은 명시적으로 구분되는 경계를 가져서 섞이지 않도록 해야한다.               
여러 하위 도메인 모델이 섞이기 시작하면 모델의 의미가 약해질 뿐만 아니라               
각 하위 도메인별로 다르게 발전하는 요구사항을 모델에 반영하기 어려워진다.                        
(그래도 논리적으로 같은 클래스가 여러개이므로 헷갈림을 자초할 수도 있기 때문이다.)        
          
모델은 **특정 컨텍스트 안에서 완전한 의미를 갖는다.**             
이렇게 **특정 컨텍스트별로 구분되는 경계를 DDD 에서는 BOUNDED CONTEXT라고 부른다.**      
     
# BOUNDED CONTEXT    
BOUNDED CONTEXT는 모델의 **한 경계를 결정**하며         
한 개의 BOUNDED CONTEXT는 논리적으로 **한 개의 모델을 갖는다.**          
              
**`BOUNDED CONTEXT`는 용어를 기준으로 구분하고 분리하고 경계를 나눈다.(카탈로그/재고/도메인마다?)**      
그리고 `BOUNDED CONTEXT`는 실제 사용자에게 기능을 제공하는 물리적인 시스템으로    
도메인 모델은 이 `BOUNDED CONTEXT` 안에서 도메인을 구현하도록 한다.     
  
[#](#)     
   
**주문 하위 도메인**       
* 주문 처리팀 : 주문 BOUNDED CONTEXT        
* 결제 처리팀 : 결제 금액 계산 BOUNDED CONTEXT       
* 카탈로그와 재고 관리는 아직 명확히 안 나뉘어져 상품 BOUNDED CONTEXT에서 구현     

이상적으로 **도메인과 BOUNDED CONTEXT가 일대일 관계**를 가지면 좋지만 대부분 그렇지 않다.            
BOUNDED CONTEXT는 기업의 팀 조직 구조에 따라 결정되기 때문이다.
               
**규모가 작은 기업의 경우 하나의 팀에서 `회원`/`주문`/`결제`/`재고`등의 도메인을 처리한다.**           
**즉, 여러 하위 도메인을 하나의 BOUNDED CONTEXT에서 구현한다.**                               
이때, 한 가지 주의점이 있는데 하위 도메인의 모델들이 뒤섞이지 않게 해야 하는 것이다.               
         
한 개의 프로젝트에서 논리적으로 비슷한 모델이 있으면       
전체 하위 도메인을 위한 단일 모델을 만들고 싶은 유혹에 빠지기 쉽다.                
이런 유혹에 걸려들면 단일 모델은 **각각의 고유한 기능을 담지 못하거나 너무 많은 기능을 가지고 있게 된다.**                   
결과적으로 **개별 하위 도메인 모델을 제대로 반영하지 못해서 하위 도메인별 기능 확장을 어렵게 만든다.**                      
**(즉, 공통된 도메인 모델을 사용하려다 보면 각 BOUNDED CONTEXT마다의 고유한 기능을 사용못하게 된다.)**           
           
따라서 한 개의 BOUNDED CONTEXT에서 여러 하위 도메인을 포함하더라도    
하위 도메인마다 구분되는 패키지를 갖도로 구현해야 한다.          
그러면 하위 도메인을 위한 모델이 서로 뒤섞이지 않아서      
하위 도메인마다 BOUNDED CONTEXT를 갖는 효과를 낼 수 있다.            
     
[#](#)     
   
BOUNDED CONTEXT는 **도메인 모델을 구분하는 경계**가 되기 때문에         
BOUNDED CONTEXT는 구현하는 **하위 도메인에 알맞은 모델을 포함한다.**      
      
* **회원 도메인 :** 회원
* **주문 도메인 :** 주문자
* **배송 도메인 :** 보내는 사람 
   
논리적으로 같은 개념이라고 할지라도       
`BOUNDED CONTEXT`에 따라 갖는 모델이 달라지며      
이들은 각 컨텍스트에 최적화 되어있는 형태를 가지고 있다.       

[#](#)  

회원의 Member는 애그리거트 루트이지만 주문의 Order는 벨류가 되고        
카탈로그의 Product는 상품이 속할 Category와 연관을 갖지만         
재고의 Product는 카탈로의 Categorty와 아무런 연관을 갖지 않는다.   

# BOUNDED CONTEXT 구현  
> BOUNDED CONTEXT는 모듈이 아니다.   
     
BOUNDED CONTEXT가 도메인 모델만 포함하는 것은 아니다.             
BOUNDED CONTEXT는 도메인 및 도메인 기능을 제공하는 표현/응용/인프라 영역 모두를 포함한다.          
도메인 모델의 데이터 구조가 바뀌면 DB 테이블 스키마도 함께 변경해야 하므로 해당 테이블도 BOUNDED CONTEXT에 포함된다.       

[#](#)   

표현 영역은 인간 사용자를 위해 HTML 페이지를 생성할 수도 있고     
다른 BOUNDED CONTEXT를 위해 REST API를 제공해줄 수 있다.      
   
모든 BOUNDED CONTEXT를 반드시 도메인 주도로 개발할 필요는 없다.       
`상품 리뷰`는 복잡한 도메인 로직을 갖지 않기 때문에 CRUD 방식으로 구현해도 된다.        
즉, DAO와 데이터 중심의 밸류 객체를 이용해서 리뷰 기능을 구현해도 기능을 유지보수 하는데 큰 문제가 없다.   
   
[#](#)      
      
서비스 DAO 구조를 사용하면 도메인 기능이 서비스에 흩어지게 되지만      
도메인 기능 자체가 단순하면 서비스-DAO로 구성된 CRUD 방식을 사용해도 코드를 유지보수하는데 문제되지 않는다.       
    
한 BOUNDED CONTEXT에서 두 방식을 혼합해서 사용할 수도 있다.       
대표적인 예가 **CQRS(Command Query Responsibility Segregation**이다.              
**CQRS 는 상태를 변경하는 명령 기능과 내용을 조회하는 쿼리 기능을 위한 모델을 구분하는 패턴이다.**       

[#](#)
   
이 패턴을 **단일 BOUNDED CONTEXT**에 적용하면        
상태 변경과 관련된 기능은 `도메인 모델` 기반으로 구현하고        
조회 기능은 `서비스-DAO`를 이용해서 구현할 수 있다.    
  
각 BOUNDED CONTEXT는 서로 다른 구현 기술을 사용할수도 있다.      
어느 BOUNDED CONTEXT는 MVC를 사용한다면 다른 BOUNDED CONTEXT는 WebFlux를 사용하거나         
MySQL 기반의 JPA 대신에 Redis, MongoDB 와 같은 NoSQL도 사용할 수 있다.     
   
BOUNDED CONTEXT는 사용자에게 보여지는 UI를 가져야하는 것은 아니다.        
BOUNDED CONTEXT의 REST API를 직접 호출해서 로딩한 JSON 데이터를 알맞게 가공해서 리뷰 목록을 보여줄수도 있다.     

[#](#)     
  
UI를 처리하는 서버를 두고 UI에서 BOUNDED CONTEXT와 통신해서 사용자 요청을 처리하는 방법도 있다.   
  
[#](#)    
  
UI 서버는 각 BOUNDED CONTEXT를 위한 파사드 역할을 수행한다.       
브라우저가 UI 서버에 요청을 보내면 UI서버는          
카탈로그와 리뷰 BOUNDED CONTEXT로부터 필요한 정보를 읽어와 조합한 뒤 브라우저에 응답을 제공한다.       
각 BOUNDED CONTEXT는 UI 서버와 통신하기 위해 HTTP, Protobuf, Thrift와 같은 방식을 이용할 수 있다.   
       
# BOUNDED CONTEXT 간 통합    
> 매출 증대를 위해 카탈로그 하위 도메인에 개인화 추천 기능을 도입하기로 했다고 가정한다.           
      
기존 카탈로그 시스템을 개발하던 팀과 별도로           
추천 시스템을 담당하는 팀이 새로 생겨서 이 팀에서 추천 시스템을 만들기로 했다.      
     
이렇게 되면 카탈로그 하위 도메인에는           
기존 카탈로그를 위한 BOUNDED CONTEXT와 추천 기능을 위한 BOUNDED CONTEXT가 생긴다.       
두 팀이 관련된 BOUNDED CONTEXT를 개발하면 자연스럽게 두 BOUNDED CONTEXT간 통합이 발생한다.           
       
카탈로그와 추천 BOUNDED CONTEXT간 통합이 필요한 기능은 다음과 같다.           
* 사용자가 제품 상세 페이지를 볼 때, 보고 있는 상품과 유사한 상품 목록을 하단에 보여준다.     
       
카탈로그 BOUNDED CONTEXT에 추천 제품 목록을 요청하면             
카탈로그 BOUNDED CONTEXT는 추천 BOUNDED CONTEXT로부터 추천 정보를 읽어와 추천 제품 목록을 제공한다.        
이때 카탈로그/추천 컨텍스트의 도메인 모델은 서로 다르다.      
카탈로그는 제품을 중심으로 도메인 모델을 구현하지만 추천은 추천 연산을 위해 모델을 구현한다.       

* 추천 : 상품의 간략한 정보만, 상품 번호 대신 아이템 ID 라는 이름을 사용
* 카탈로그 : 추천 데이터는 받아오지만 도메인을 활용한다기 보다는 도메인 모델을 사용해서 추천 상픔 표현   

```java
/ *
  * 상품 추천 기능을 표현하는 도메인 서비스(카탈로그 기반 추천 이용)    
  */
public interface ProductRecommnedationService {
    public List<Product> getRecommendationOf(ProductId id);
}
```
  
도메인 서비스를 구현한 클래스는 인프라스트럭처 영역에 위치한다.       
이 클래스는 외부 시스템과의 연동을 처리하고 외부 시스템의 모델과 현재 도메인 모델간의 변환을 책임진다.     

[#](#)  

RecSystemClient는 외부 추천 시스템이 제공하는 REST API를 이용해서          
특정 상품을 위한 추천 상품 목록을 로딩한다.       
이 REST API가 제공하는 데이터는 추천 시스템의 모델을 기반으로 하고 있기 때문에          
API 응답은 다음과 같이 상품 도메인 모델과 일치하지 않는 데이터를 제공할 것이다.       

```json
[
    {itemId: 'PROD-1000', type: 'PRODUCT', rank: 100},
    {itemId: 'PROD-1001', type: 'PRODUCT', rank: 54}   
]
```
RecSystemClient는 REST API로부터 데이터를 읽어와 카탈로그 도메인에 맞는 상품 모델로 변환한다.     

```java
public class RecSystemClient implements ProductRecommendationService {
    private ProductRepository productRepository;
    
    @Override
    public List<Product> getRecommendationsOf(ProductId id) {
        List<RecommendationItem> items = getRecItems(id.getValue());
        return toProducts(items);
    } 
    
    @Override
    public List<RecommendationItem> getRecItems(String ) {
        // externalRecClient는 외부 추천 시스템을 위한 클라이언트라고 가정 
        return externalRecClient.getRecs(itemId);
    }
    
    private List<Product> toProducts(List<RecommendationItem> items) {
        return items.stream()
            .map(item -> toProductId(item.getItemId()))
            .map(prodId -> productRepository.findById(prodId))
            .collect(toList());
    }
    
    private ProductId toProductId(String itemId) {
        return new ProductId(itemId);
    }
    ...
}
```    
`getRecItems()`메서드에서 사용하는 `externalRecClient`는            
외부 추천 시스템에 연결할 때 사용하는 클라이언트로서           
추천 시스템을 관리하는 팀에서 배포하는 모듈이라고 가정하자       




     



   




















