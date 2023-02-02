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
}
```

→ ManyToOne 방식은 기본이 eager 방식이여서 Lazy 방식으로 처리 필요

→ EnumType 열거형 사용

→ CascadeType 이란?(연관관계 관련 처리)

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
}
```

→ ManyToOne 방식으로 item과 order에 연관 처리

→ ManyToOne 은 기본적으로 즉시 로딩이여서 지연 로딩으로 처리해야함

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

**회원 엔티티 기능**

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

상품 **엔티티 기능**

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
