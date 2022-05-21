# 사용자가 결제 버튼을 누르고 이후의 FLOW

## 결제완료

### OrderService

```java
public class OrderService{
	@Transactional
	public void payOrder(Long orderId){
		Order order = orderRepository.findById(orderId)...
		order.payed(); // <--- 요청!

		Delivery delivery = Delivery.started(order);
		deliverRepository.save(delivery);
	}
}
```

### Order

```java
public class Order {
	public enum OrderStatus{
		ORDERED, PAYED, DELIVERED
	}
	
	@Enumerated(EnumType.String)
	@Column(name="STATUS")
	private OrderStatus orderStatus;

	public void payed(){
		this.orderStatus = PAYED;
	}
}
```

### Delivery

```java
@Entity
@Table(name = "DELIVERIES)
public class Delivery {
	enum DeliveryStatus{ DELIVERING, DELIVERED }

	@OneToOne
	@JoinColumn(name = "ORDER_ID")
	private Order order;

	@Enumerated(EnumType.STRING)
	@Column(name = "STATUS")
	private DeliveryStatus deliveryStatus;

	**public static Delivery started(Order order){
		return new Delivery(order.DELIVERING);
	}

	// 배달완료
	public void complete(){
		this.deliveryStatus = DELIVERED;
		this.order.completed();
	}**
}
```

## 배달완료

### OrderService

```java
public class OrderService{
	
	@Trancationsl
	public void deliverOrder(Long orderId){
		Order order = orderRepository.findById(orderId)...
		**order.delivered();**

		Deliver delivery = deliveryRepository.findById(orderId);
		**delivery.complete();**
	}
}
```

1. order → shop
2. delivery → order

### Order

```java
public class Order {
	public void delivered(){
		this.orderStatus = DELIVERED;
		this.shop.billCommissionFee(calculateTotalPrice());
	}
}
```

### Shop

```java
pubic class Shop{
	private Ratic commissionRate;
	private Money commission = Money.ZERO;

	public void billCommissionFee(MoneyPrice){
		commission = commission.plus(commissionRate.of(price));
	}
}
```

![스크린샷 2022-05-21 오후 2.09.26.png](%E1%84%89%E1%85%A1%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%8C%E1%85%A1%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%A7%E1%86%AF%E1%84%8C%E1%85%A6%20%E1%84%87%E1%85%A5%E1%84%90%E1%85%B3%E1%86%AB%E1%84%8B%E1%85%B3%E1%86%AF%20%E1%84%82%E1%85%AE%E1%84%85%E1%85%B3%E1%84%80%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%AE%E1%84%8B%E1%85%B4%20FLOW%20ac404fb3400f4bf3895c5f9cbcfbe1d8/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-05-21_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_2.09.26.png)

- 무엇이 문제일까?
    - delivery (배달) 의 complete() 메서드가 호출되면 여러 객체들의 협력을 요구한다.
        - delivery → order
        - order → shop
            
            ![스크린샷 2022-05-21 오후 2.14.34.png](%E1%84%89%E1%85%A1%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%8C%E1%85%A1%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%A7%E1%86%AF%E1%84%8C%E1%85%A6%20%E1%84%87%E1%85%A5%E1%84%90%E1%85%B3%E1%86%AB%E1%84%8B%E1%85%B3%E1%86%AF%20%E1%84%82%E1%85%AE%E1%84%85%E1%85%B3%E1%84%80%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%AE%E1%84%8B%E1%85%B4%20FLOW%20ac404fb3400f4bf3895c5f9cbcfbe1d8/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-05-21_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_2.14.34.png)
            
        - long transation 으로 묶어있지만, 트랜젝션으로 물러있는 주기가 다르다.
        - 실제사례
            - 사용자에게 노출되는 로직이 다 에러로 떨어짐. 살펴보니 다 락이 걸려있다.
            - 운영하는 쪽에서 대용량 데이터를 업데이트 치는 작업을 처버림. 그래서 트랜잭션안에서 락이 전부 다 걸림.. 그래서 요청왔을 때 전부 튕겨나감
            
            ![스크린샷 2022-05-21 오후 2.17.31.png](%E1%84%89%E1%85%A1%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%8C%E1%85%A1%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%A7%E1%86%AF%E1%84%8C%E1%85%A6%20%E1%84%87%E1%85%A5%E1%84%90%E1%85%B3%E1%86%AB%E1%84%8B%E1%85%B3%E1%86%AF%20%E1%84%82%E1%85%AE%E1%84%85%E1%85%B3%E1%84%80%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%AE%E1%84%8B%E1%85%B4%20FLOW%20ac404fb3400f4bf3895c5f9cbcfbe1d8/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-05-21_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_2.17.31.png)
            
            - 객체들을 쫗아가면서 수정하다 보면 트랜잭션 경합으로 인해 성능 저하 또는 응답성성에 대해 성능 떨어 질 수 밖에 없다..

### 객체 참조의 문제점

- *“Everythine is connected”*
- 모든 객체가 연결돼 있기 때문에 어떤 객체라도 접근가능, 어떤 객체라도 함께 수정 가능
- ***객체 참조는 결합도가 가장 높은 의존성, 필요한 경우에는 객체 참조를 끊을 수 있어야한다.***
- 객체참조를 통한 탐색(강한 결합도)
    
    ```java
    // ORDER
    @Entity
    @Table(name = "ORDERS")
    public class Order{
    	@ManyToOne
    	@JoinColumn(name = "SHOP_ID")
    	private Shop shop;
    }
    
    // SHOP
    @Entiy
    @Table(name = "SHOPS")
    public class Shop{
    	@Id
    	@GeneratedValue(strategy = GenerationType.IDENTITY)
    	@Column(name = "SHOP_ID")
    	private Long id;
    }
    ```
    
- Repository를 통한 탐색(약한 결합도) : 결합도를 낮추면서 연관관계를 구현하는 방법
    
    ```java
    @Entity
    @Table(name = "ORDERS")
    public class Order{
    	
    	@Column(name = "SHOP_ID")
    	private Long shopId;
    }
    
    @Entiy
    @Table(name = "SHOPS")
    public class Shop{
    	@Id
    	@GeneratedValue(strategy = GenerationType.IDENTITY)
    	@Column(name = "SHOP_ID")
    	private Long id;
    ```
    
    ```java
    Order order = orderRepository.findById(orderId);
    Shop shop = shopRepository.findById(order.getShopId());
    ```