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
