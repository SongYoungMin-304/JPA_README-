JPA_README 정리

## JPA 란?

- Java Persistence Api  의 약자
- 객체지향적으로 프로그램을 만들 수 있도록 도와 주는 기술

  > 관계형 DB(RDB) 와 연결을 할 때 RDB와 맞춰서 직접 쿼리를 작성함

  > 객체 지향적으로 객체만 생성하거나 수정하면 쿼리 등을 자동으로 실행해주는 API

     (MYBATIS 등 처리 직접 쿼리를 생성해 DB에 날리지 않고 객체를 DB에 맞춰서 설계한 이후
      객체를 통해 쿼리를 실행한다.)

![1](https://user-images.githubusercontent.com/56577599/215508374-bd8e41ab-cc40-4f02-a51e-7169389f3caf.png)


## 도메인 모델과 테이블 설계

  **도메인 설계**
  
![2](https://user-images.githubusercontent.com/56577599/215508404-8a6b8c87-fac7-4876-867e-0477a0cacb62.png)


  **객체 설계**
  
![3](https://user-images.githubusercontent.com/56577599/215508448-61d68ef0-7ee7-40d7-8ee3-4fd3f6546445.png)



  **테이블 설계**
  
  ![4](https://user-images.githubusercontent.com/56577599/215508507-b44b9711-a937-4045-8b1d-1b6bfe9b0292.png)

  `⭐` 엔티티를 설계할 때 주의 할 점
  
  
- one to many 단계에서는 many 가 연관관계의 **주인**
- one to one 또는 many to many 에서는 외래키가 존재하는 부분이 **주인**

→ 주인이면 @JoinColumn(name=””), 종속이면(mappedBy =””) 로 처리
→ 주인이라고 테이블의 부모 값을 의미하는 건 아님 오히려 부모는 Member

**엔티티 설계**

**회원**

```java
public class Member {

 @Id @GeneratedValue
 @Column(name = "member_id")
 private Long id;
 
 private String name;
 
 @Embedded
 private Address address;
 
 @OneToMany(mappedBy = "member")
 private List<Order> orders = new ArrayList<>();

}
```

→ @Embedded 컬럼 축약 정도의 의미

→ order 를 향해 one to many로 처리

**주소**

```java
@Embeddable
@Getter
public class Address {
     private String city;
     private String street;
     private String zipcode;

     protected Address() {
     }

 public Address(String city, String street, String zipcode) {
     this.city = city;
     this.street = street;
     this.zipcode = zipcode;
 }
}
```

→ Embeddable 은 protected 생성자 존재해야함

**주문**

```java
@Entity
@Table(name = "orders")
@Getter @Setter
public class Order {

      @Id @GeneratedValue
      @Column(name = "order_id")
      private Long id;
 
     @ManyToOne(fetch = FetchType.LAZY)
     @JoinColumn(name = "member_id")
     private Member member; //주문 회원
   
     @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
     private List<OrderItem> orderItems = new ArrayList<>();
 
     @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
     @JoinColumn(name = "delivery_id")
     private Delivery delivery; //배송정보
 
     private LocalDateTime orderDate; //주문시간
 
     @Enumerated(EnumType.STRING)
     private OrderStatus status; //주문상태 [ORDER, CANCEL]
 
 //==연관관계 메서드==//
     public void setMember(Member member) {
         this.member = member;
         member.getOrders().add(this);
     }
 
     public void addOrderItem(OrderItem orderItem) {
         orderItems.add(orderItem);
         orderItem.setOrder(this);
     }

     public void setDelivery(Delivery delivery) {
         this.delivery = delivery;
         delivery.setOrder(this);
     }

     //==생성 메서드==//
     public static createOrder(Member member, Delivery delivery, OrderItem... orderItems){
          Order order = new Order();
          order.setMember(member);
          order.setDelivery(delivery);
          for(OrderItem orderItem : orderItems){
             order.addOrderItem(orderItem);
          }
          order.setStatus(OrderStatus.ORDER);
          order.setOrderDate(LocalDateTeim.now());
          return order;
     }

     //==비즈니스 로직==//
     /** 주문 취소 */
     public void cancel(){
          if(delivery.getStatus() == DeliveryStatus.COMP){
                 throw new IllegalStateException("이미 배송완료된 상품은 취소가 불가능합니다.");
          }
          
          this.setStatus(OrderStatus.CANCEL);
          for(OrderItem orderItem : orderItems){
            orderItem.cancel();
          }
        }
  
     public int getTotalPrice(){
         int totalPrice = 0;
         for(OrderItem orderItem : orderItems){
             totalPrice += orderItem.getTotalPrice();
         }
         return totalPrice;
      }
         
}
```

→ ManyToOne 방식은 기본이 eager 방식이여서 Lazy 방식으로 처리 필요

→ EnumType 열거형 사용

### **생성 메소드(createOrder)**

**→ order에다가 member, delivery, orderItem 세팅, status, orderDate 값 세팅**
**(orderItem 이랑 delivery 값이 cascade 로 세팅 되어있기 떄문에 order를 저장하면 orderItem, delivery 또한 저장됨)**

### 주문 취소**(**cancel**)**

**→ order에다가 상태값 세팅, orderItem cancel 로직 적용**
**(orderItem cancel 로직은 아래 내용 확인)**

### 총 금액 합계 구하기**(getTotalPrice)**

**→ orderItem for문 돌리면서 금액 합쳐서 return** 

## ** CascadeType 이란?(연관관계 관련 처리)

```java
Order order = ~~~
Delivery delivery = ~~~

order.setDelivery(delivery);

em.persist(order);
--> cascade 가 처리되어 있으면 order 생성 및 delivery 를 생성한다.
--> cascade 가 처리되어 있지 않으면 에러 발생
--> 이런 경우

Order order = ~~~
Delivery delivery = ~~~

em.persis(delivery);

order.setDelivery(delivery);

em.persist(order); 
--> casecade 없이도 에러 나지 않음
```

**주문상태**

```java
public enum OrderStatus {
      ORDER, CANCEL 
}
```

**주문상품 엔티티**

```java
@Entity
@Table(name = "order_item")
@Getter @Setter
public class OrderItem {
 
    @Id @GeneratedValue
    @Column(name = "order_item_id")
    private Long id;
 
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "item_id")
    private Item item; //주문 상품
 
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order; //주문
 
    private int orderPrice; //주문 가격
    private int count; //주문 수량

    //==생성 메서드==//
    public static OrderItem createOrderItem(Item item, int orderPrice, int count){
         OrderItem orderItem = new OrderItem();
         orderItem.setItem(item);
         orderItem.setOrderPrice(orderPrice);
         orderItem.setCount(count);

         item.removeStock(count);
         return orderItem;
         }

    //==비즈니스 로직==//
    /** 주문 취소 */
    public void cancel(){
        getItem().addStock(count);
    }

    //==조회로직==//
    public int getTotalPrice(){
         return getOrderPrice() * getCount();
    }

}
```

→ ManyToOne 방식으로 item과 order에 연관 처리

→ ManyToOne 은 기본적으로 즉시 로딩이여서 지연 로딩으로 처리해야함

### **생성 메소드(createOrderItem)**

**→ orderItem 에다가 item 세팅해주고 금액과 갯수를 세팅 item 의 재고를 감소 시켜준다.**

### 주문 취소**(**cancel**)**

**→ item을 가져와서 재고를 더해준다.**

### 조회 로직**(getTotalPrice)**

**→ orderPrice 를 가져오고 count를 가져와서 곱한다…**

**상품 엔티티**

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype")
@Getter @Setter
public abstract class Item {

     @Id @GeneratedValue
     @Column(name = "item_id")
     private Long id;
 
     private String name;
 
     private int price;
 
     private int stockQuantity;
 
     @ManyToMany(mappedBy = "items")
     private List<Category> categories = new ArrayList<Category>();

     //==비즈니스 로직==//
     public void addStock(int quantity) {
        this.stockQuantity += quantity;
     }
 
     public void removeStock(int quantity) {
        int restStock = this.stockQuantity - quantity;
 
        if (restStock < 0) {
           throw new NotEnoughStockException("need more stock");
        }
        
        this.stockQuantity = restStock;
     }
}
```

- 연관관계 메소드를 통해서 엔티티의 재고를 변경
- 재고의 숫자를 늘리기 및 내리기 처리 가능

**상품 - 도서 엔티티**

```java
@Entity
@DiscriminatorValue("B")
@Getter @Setter
public class Book extends Item {
      private String author;
      private String isbn;
}
```

**상품 - 음반 엔티티**

```java
@Entity
@DiscriminatorValue("A")
@Getter @Setter
public class Album extends Item {
      private String artist;
      private String etc;
}
```

**상품 - 영화 엔티티**

```java
@Entity
@DiscriminatorValue("M")
@Getter @Setter
public class Movie extends Item {
      private String director;
      private String actor;
}
```

@Inheritance 방식

1) SINGLE_TABLE 

```sql
create table Item (
      DTYPE varchar(31) not null,
      id bigint generated by default as identity,
      name varchar(255),
      price integer not null,
      artist varchar(255),
      author varchar(255),
      isbn varchar(255),
      actor varchar(255),
      director varchar(255),
      primary key (id)
  )
```

2) JOINED

```sql
create table Album (
      artist varchar(255),
      id bigint not null,
      primary key (id)
  )

create table Book (
      author varchar(255),
      isbn varchar(255),
      id bigint not null,
      primary key (id)

create table Movie (
      actor varchar(255),
      director varchar(255),
      id bigint not null,
      primary key (id)

create table Item (
      DTYPE varchar(31) not null,
      id bigint generated by default as identity,
      name varchar(255),
      price integer not null,
      primary key (id)

alter table Album
      add constraint FKcve1ph6vw9ihye8rbk26h5jm9
      foreign key (id)
      references Item

alter table Book
      add constraint FKbwwc3a7ch631uyv1b5o9tvysi
      foreign key (id)
      references Item

add constraint FK5sq6d5agrc34ithpdfs0umo9g
      foreign key (id)
      references Item
```

3) TABLEPERCLASS

```sql
create table Album (
      id bigint not null,
       name varchar(255),
       price integer not null,
       artist varchar(255),
       primary key (id)

create table Book (
      id bigint not null,
       name varchar(255),
       price integer not null,
       author varchar(255),
       isbn varchar(255),
       primary key (id)

create table Movie (
      id bigint not null,
       name varchar(255),
       price integer not null,
       actor varchar(255),
       director varchar(255),
       primary key (id)
```

- 하위 테이블 없이 각 테이블을 전부 생성

→ @DiscriminatorColumn(name = "dtype")
: 상위 테이블 컬럼 타입

→ @DiscriminatorValue(”M”)
: 구분 값 세팅 

**배송**

```java
@Entity
@Getter @Setter
public class Delivery {
     @Id @GeneratedValue
     @Column(name = "delivery_id")
     private Long id;
 
     @OneToOne(mappedBy = "delivery", fetch = FetchType.LAZY)
     private Order order;
 
     @Embedded
     private Address address;
 
     @Enumerated(EnumType.STRING)
     private DeliveryStatus status; //ENUM [READY(준비), COMP(배송)]
}
```

**배송 상태**

```java
public enum DeliveryStatus {
    READY, COMP
}
```

**카테고리 엔티티**

```java
@Entity
@Getter @Setter
public class Category {
     @Id @GeneratedValue
     @Column(name = "category_id")
     private Long id;
 
     private String name;
 
     @ManyToMany
     @JoinTable(name = "category_item",
     joinColumns = @JoinColumn(name = "category_id"),
     inverseJoinColumns = @JoinColumn(name = "item_id"))
     private List<Item> items = new ArrayList<>();
 
     @ManyToOne(fetch = FetchType.LAZY)
     @JoinColumn(name = "parent_id")
     private Category parent;
 
     @OneToMany(mappedBy = "parent")
     private List<Category> child = new ArrayList<>();
 
 //==연관관계 메서드==//
 public void addChildCategory(Category child) {
     this.child.add(child);
     child.setParent(this);
 }
}
```

→ ManytoMany 방식 사용(실무에서는 비추)
 - @JoinTable(name =”category_item”)으로 조인 테이블 세팅

 -  joinColumns = @JoinColumn(name = "category_id"),
    inverseJoinColumns = @JoinColumn(name = "item_id"))
—> 이걸 통해서 category 와 category_item 간에 연관 컬럼 처림

→ parent 를 통해서 하나의 부모를 가지고 여러개의 자식을 가질 수 있도록 설정

→ 연관관계 메서드

1) 부모에 세팅해 주고

2) 자식에 추가해준다.

### 엔티티 설계시 주의 할 점

1) 모든 연관관계는 지연로딩으로 설정
2) 컬렉션은 필드에서 바로 초기화 하는 것이 안전한다.

```java
Member member = new Member();
System.out.println(member.getOrders().getClass());
em.persist(member);
System.out.println(member.getOrders().getClass());

//출력 결과
class java.util.ArrayList
class org.hibernate.collection.internal.PersistentBag
```

`⭐` 지연 로딩 시 주의 할 점

- 연관 관계에서는 연관관계의 주인이 필요하다. 일반적으로 외래키를 가지고 있는 테이블(엔티티)가 연관관계의 주인이다.

Delivery 엔티티

```java
 public class Delivery {

    @Id
    @GeneratedValue
    @Column(name ="delivery_id")
    private Long id;

    @JsonIgnore
    @OneToOne(mappedBy = "delivery", fetch = LAZY)
    private Order order;
}
```

Order 엔티티

```java
public class Order {

    @Id @GeneratedValue
    @Column(name ="order_id")
    private Long id;

    @OneToOne(fetch = LAZY) // cascade 로 order 가 저장될때 orderItem 도 같이 저장됨... 라이프 사이클에서 delivery// 을 건드는 경우가 order 만 인 경우
    @JoinColumn(name ="delivery_id")
    private Delivery delivery;
}
```

→ em.find(Delivery.class, 3L) 등으로 엔티티를 가져올 때 

DELIVERY 를 가져오는 쿼리  + Order 를 가져오는 쿼리 두번이 실행된다.

→ em.find(Order.class, 3L) 등으로 엔티티를 가져올 때 

 Order 를 가져오는 쿼리 한번만 실행된다.

** 이유는 

→ 우선 DB 상으로 볼때 

```sql
create table delivery (
        delivery_id number(19,0) not null,
        primary key (delivery_id)
    )

create table orders (
        order_id number(19,0) not null,
        member_id number(19,0),
        primary key (order_id)
    )

alter table orders 
       add constraint FKtkrur7wg4d8ax0pwgo0vmy20c 
       foreign key (delivery_id) 
       references delivery
```

→ ORDER는 무조건 delivery 를 가지고 있다는 것을 의미한다.
즉 **order가 있는 데 delivery 가 없을 수 는 없기 때문에 delivery 엔티티가 있다는 것이 보장되**기 때문에 proxy로 처리할 수 가 있다.

### **—> 즉 지연로딩이 가능하다는 것이다.**

→ delivery 가 있다고 무조건 order 가 있다는 것을 의미하지 않기 때문에(db 상에서 문제가 없음) 

proxy로 처리할 수 가 없기 때문에 (null 일 수도 있어서)  

### **—> 즉 지연로딩이 처리가 안되고 즉시 연동처리가 됨**

### 그러면 모든 주인이 아닌 엔티티에서는 주인 엔티티를 조회 할 수 없는가?

### 결론은 아니다.!  oneToMany 경우에서는 List 객체를 가지고 있기 때문에 null 이 아니라고 판단하고 지연로딩이 된다.

## **즉 1대1관계에서 주인이 아닌 경우에는 왠만하면 단방향으로 설계하는 것이 좋다.**

---

## 기능 구현

**회원 기능**

```sql
@Repository
public class MemberRepository {

 @PersistenceContext

     private EntityManager em;

     public void save(Member member) {
        em.persist(member);
     }

     public Member findOne(Long id) {
        return em.find(Member.class, id);
     }

     public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class)
               .getResultList();
     }

     public List<Member> findByName(String name) {
        return em.createQuery("select m from Member m where m.name = :name",
              Member.class)
             .setParameter("name", name)
             .getResultList();
 }
}
```

→ persist 기능 - 저장

→ find 기능 - 조회 기능

→ createQuery - jpql을 통해서 쿼리 생성해서 키 값 말고도 조회 가능

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {
 
     private final MemberRepository memberRepository;
     /**
      * 회원가입
      */
 
     @Transactional //변경
     public Long join(Member member) {
         validateDuplicateMember(member); //중복 회원 검증
         memberRepository.save(member);
         return member.getId();
     }
     
     private void validateDuplicateMember(Member member) {
         List<Member> findMembers =
         memberRepository.findByName(member.getName());
 
        if (!findMembers.isEmpty()) {
             throw new IllegalStateException("이미 존재하는 회원입니다.");
         }
     }
     /**
     * 전체 회원 조회
     */
 public List<Member> findMembers() {
         return memberRepository.findAll();
 }
 
 public Member findOne(Long memberId) {
         return memberRepository.findOne(memberId);
 }

}
```

**상품 기능**

```sql
@Repository
@RequiredArgsConstructor
public class ItemRepository {

      private final EntityManager em;
      
      public void save(Item item) {
        if (item.getId() == null) {
                 em.persist(item);
        } else {
                 em.merge(item);
        }
        }

      public Item findOne(Long id) {
          return em.find(Item.class, id);
        }

      public List<Item> findAll() {
         return em.createQuery("select i from Item i",Item.class).getResultList();
 }
}
```

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class ItemService {
 
    private final ItemRepository itemRepository;
 
    @Transactional
    public void saveItem(Item item) {
       itemRepository.save(item);
    }
 
    public List<Item> findItems() {
       return itemRepository.findAll();
    }
 
    public Item findOne(Long itemId) {
       return itemRepository.findOne(itemId);
    }
}
```

**주문 기능**

```java
@Repository
@RequiredArgsConstructor
public class OrderRepository {

     private final EntityManager em;
     
    public void save(Order order) {
         em.persist(order);
     }

     public Order findOne(Long id) {
         return em.find(Order.class, id);
     }

// public List<Order> findAll(OrderSearch orderSearch) { ... }
}
```

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderService {

     private final MemberRepository memberRepository;
     private final OrderRepository orderRepository;
     private final ItemRepository itemRepository;
 
 /** 주문 */
 @Transactional
 public Long order(Long memberId, Long itemId, int count) {
     //엔티티 조회
     Member member = memberRepository.findOne(memberId);
     Item item = itemRepository.findOne(itemId);
 
     //배송정보 생성
     Delivery delivery = new Delivery();
     delivery.setAddress(member.getAddress());
     delivery.setStatus(DeliveryStatus.READY);
 
     //주문상품 생성
     OrderItem orderItem = OrderItem.createOrderItem(item, item.getPrice(),
count);
     
     //주문 생성
     Order order = Order.createOrder(member, delivery, orderItem);
 
     //주문 저장
     orderRepository.save(order);
     return order.getId();
 }
 
 /** 주문 취소 */
 @Transactional
 public void cancelOrder(Long orderId) {
     //주문 엔티티 조회
     Order order = orderRepository.findOne(orderId);
     //주문 취소
     order.cancel();
 }
 /** 주문 검색 */
/*
 public List<Order> findOrders(OrderSearch orderSearch) {
 return orderRepository.findAll(orderSearch);
 }
*/
}
```

### 주문 **생성 로직**

1) memberId를 통해서 회원을 가져옴

2) itemId를 통해서 아이템을 가져옴

3) 배송 정보 세팅

4) 주문상품 데이터 생성 OrderItem 비즈니스 로직 참조

5) 주문생성 회원과 배송지와 orderItem 을 가지고 주문 생성

(cascade로 delivery 와 orderItem 생성됨)

### 주문 취소 **로직**

1) 주문을 가져옴

2) 주문의 비즈니스 취소 로직을 적용





## JPA API 개발 시 성능 최적화 방식

### 회원생성

### V1

```java
@Data
static class CreateMemberRequest {
    private String name;
}

@Data
static class CreateMemberResponse {
    private Long id;

    public CreateMemberResponse(Long id) {
       this.id = id;
    }
}

@PostMapping("/api/v1/members")
public CreateMemberReponse saveMemberV1(@RequestBody @Valid Member member
{
     Long id = memberService.join(member);
     return new CreateMemberResponse(id);
}

```

→ v1 : 엔티리를 Request Body에 직접 매핑

- 엔티티에 API 검증을 위한 로직이 들어간다(@NotEmpty)
- 한 엔티티에 각각의 api를 위한 모든 요구사항을 담기가 어려움
- 엔티티가 변경되면 api 스펙이 변한다.

### V2

```java
@PostMapping("/api/v2/members")
 public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) 
 { 
      Member member = new Member();
      member.setName(request.getName());
      
      Long id = memberService.join(member);
      
      return new CreateMemberResponse(id);
 }
```

→ v2: 엔티티 대신에 CreateMemberRequest 를 만들어서 매핑

- 엔티티와 프레젠테이션 계층을 위한 로직 분리 가능
- 엔티티와 api 스펙을 명확하게 분리할 수 있다.
- 엔티티가 변해도 API 스펙이 변하지 않는다.

### 회원수정

```java
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id, @RequestBody @Valid UpdateMemberRequest request) {
      memberService.update(id, request.getName());
      Member findMember = memberService.findOne(id);
      return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}
 
@Data
static class UpdateMemberRequest {
      private String name;
}
 
@Data
@AllArgsConstructor
static class UpdateMemberResponse {
      private Long id;
      private String name;
}

----------------------------------------------------------------

@Transactional
public void update(Long id, String name){
     Member member = memberRepository.findOne(id);
     member.setName(name);
     }
}
```

→ DTO에 요청 파라미터를 담아서 처리한다.

→ 변경 감지를 통해 진행한다.

### 회원조회 API

### V1

```java
@GetMapping("/api/v1/members")
public List<Member> membersV1() {
     return memberService.findMembers();
}

```

→ 응답 값으로 엔티티를 외부에 노출함

→ 기본적으로 엔티티의 모든값이 노출됨

→ 여러 api에서 스펙을 엔티티에 맞추기가 어려움

→ 엔티티가 변경되면 api 스펙이 변한다.

### V2

```java
@GetMapping("/api/v2/members")
public Result membersV2(){
     List<Member> findMembers = memberService.findMembers();

     List<MemberDto> collect = findMembers.stream()
             .map(m -> new MemberDto(m.getName())))
             .collect(Collectors.toList());

     return new Result(collect);
}

@Data
@AllArgsConstructor
static class Result<T> {
    private T data;
}

@Data
@AllArgsConstructor
static class MemberDto {
    private String name;
}
```

→ 엔티티를 DTO로 변환해서 반환한다.

→ 엔티티가 변해도 API 스펙이 변경되지 않는다.

→ 추가로 Result 클래스로 컬렉션을 감싸서 향후 필요한 필드 추가가능

### 회원조회 API-고급

회원
![image](https://user-images.githubusercontent.com/56577599/218297855-59a5fa85-a895-4459-8a76-dd5b0e455fb3.png)


주문
![image](https://user-images.githubusercontent.com/56577599/218297836-a99286ce-342e-4605-9f74-8136bed3d696.png)


주문 - 아이템
![image](https://user-images.githubusercontent.com/56577599/218297816-870ba105-8542-4fe7-9848-d306df8d39b9.png)


배송
![image](https://user-images.githubusercontent.com/56577599/218297871-8141a730-5dbf-491e-a5ac-dcfb6879c04a.png)


아이템
![image](https://user-images.githubusercontent.com/56577599/218297797-c888e1bc-b39a-448a-a030-2d5f3ce13e54.png)


### V1 - 엔티티를 직접 노출

```java
@GetMapping("/api/v2/simple-orders")
public List<Order> ordersV1(){
     List<Order> all = orderRepository.findAllByString(new OrderSearch()){
     for(Order order : all{
         order.getMember().getName();
         order.getDelivery().getAddress();
     }
     return all;
}
```

→ 엔티티를 직접 노출하는 것은 좋지 않음

→ 지연로딩 처리를 통해서 데이터 전체를 가져오게 처리

→ json은 proxy객체를 어떻게 사용하는 지 몰라서 Hibernate5Module 사용해야함

(proxy를 보여주는 기능, 지연로딩 처리한거 까지 json으로 보여주는 기능 존재)

→ @JsonIgnore 한쪽은 처리해야함(양방향에서 문제 발생)

### V2 - 엔티티를 DTO로 변환

```java
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2(){
     List<Order> orders = orderRepository.findAll();
     List<SimpleOrderDto> result = orders.stream()
           .map(o -> new SimpleOrderDto(o))
           .collect(toList());
   
     return result;
}

@Data
static class SimpleOrderDto{
   
     private Long orderId;
     private String name;
     private LocalDateTime orderDate;
     private OrderStatus orderStatus;
     private Address address;

  public SimpleOrderDto(Order order){
     orderId = order.getId();
     name = order.getMember().getName();
     orderDate = order.getOrderDate();
     orderStatus - order.getStatus();
     address = order.getDelivery().getAddress();
  }
}

```

→ 엔티티를 DTO로 변환하는 일반적인 방법

→ 쿼리가 **주문 1번, 주문 횟수만큼 회원 조회, 주문횟수 만틈 배송 조회**

→ EX) 주문이 각자 다른 5명의 고객이면 쿼리가 1 + 5 + 5 총 11번 실행된다.
