# 도메인

- 문제 해결 영역
- 온라인 쇼핑몰을 예시로 들면
  - 회원, 상품 등이 될 수 있다.
  - 도메인은 하위도메인을 가질 수 있는데
    - 상품의 경우 리뷰 같은 것들이 하위도메인이 될 수 있다.
- 도메인 전문가와 대화하는 개발자, 이해관계자는 도메인 전문가 만큼은 아니지만 도메인 지식을 갖춰야 한다.

## 도메인 모델 패턴

- 일반적인 어플리케이션은 다음과 같은 4개의 영역으로 구성된다.

### 표현 (Presentation)

- 사용자 요청을 처리하고 정보를 보여줌.
- 사용자는 소프트웨어를 사용하는 사람 뿐만 아니라 외부 시스템일 수 있다.

### 응용 (Application)

- 사용자가 요청한 기능 실행
- 비즈니스 로직을 직접 구현하지 않으며 도메인 계층을 조합해서 기능 실행

### 도메인 (Domain)

- 시스템이 제공할 도메인 규칙을 구현
- 예를들면 주문의 경우 `주문 취소는 배송 전에만 할 수 있다.`와 같은 규칙이 도메인 계층에 위치한다.

### 인프라스트럭처 (Infrastructure)

- 데이터베이스나 메시징 시스템과 같은 외부 시스템과의 연동을 처리

```ts
enum OrderState {
  PAYMENT_WAITING = 1,
  PREPARING = 2,
  SHIPPED = 3,
  DELIVERING = 4,
  DELIVERY_COMPLETED = 5,
}

// 도메인 계층에 선언된 Order 클래스
class Order {
  private orderStatus: OrderState;
  private shippingInfo: ShippingInfo;

  changeShippingInfo(shippingInfo: ShippingInfo) {
    if (!this.isShippingChangeable()) {
      throw new Error(`Can't change shipping info in ${this.orderState}`);
    }
    this.shippingInfo = shippingInfo;
  }

  private isShippingChangeable(): boolean {
    return (
      this.orderState === OrderState.PAYMENT_WAITING ||
      this.orderState === OrderState.PREPARING
    );
  }
}
```

## 도메인 모델 도출

- 도메인 모델을 도출하려면 도메인에 대한 이해가 있어야한다.
- 다음과 같은 기능이 있다고 생각해보자
  - 최소 한종류 이상의 상품을 주문해야 한다.
  - 한 상품을 한 개 이상 주문할 수 있다.
  - 총 주문 금액은 각 상품의 구매 가격 합을 모두 더한 금액이다.
  - 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다.
  - 주문할 때 배송지 정보를 반드시 지정해야 한다.
  - 배송지 정보는 받는 사람 이름, 전화번호, 주소로 구성된다.
  - 추록를 하면 배송지는 변경 불가능하다.
  - 출고 전에 주문을 취소할 수 있다.
  - 고객이 결제를 완료하기 전에는 상품을 준비하지 않는다.
- 이때 주문은 `주문 상태 변경하기`, `배송지 정보 변경하기`, `주문 취소하기`와 같은 기능을 제공한다.
- 그러면 아래와 같이 메서드를 추가할 수 있다.

위 조건으로 도메인 모델을 정의해보자.

```ts
// shippingInfo/domain/model.ts
class ShippingInfo {
  private receiverName: string;
  private receiverPhoneNumber: string;
  private shippingAddress1: string;
  private shippingAddress2: string;
  private shippingZipcode: string;

  constructor(
    receiverName: string,
    receiverPhoneNumber: string,
    shippingAddress1: string,
    shippingAddress2: string,
    shippingZipcode: string
  ) {
    this.receiverName = receiverName;
    this.receiverPhoneNumber = receiverPhoneNumber;
    this.shippingAddress1 = shippingAddress1;
    this.shippingAddress2 = shippingAddress2;
    this.shippingZipcode = shippingZipcode;
  }
}

// Order/domain/model.ts
enum OrderStatus {
  PAYMENT_WAITING = 1,
  PREPARING = 2,
  SHIPPED = 3,
  DELIVERING = 4,
  DELIVERY_COMPLETED = 5,
  CANCELED = 6,
}

class Order {
  private orderLines: OrderLines[];
  private totalAmount: Money;
  private shippingInfo: ShippingInfo;
  private orderStatus: OrderStatus;

  private constructor(orderLines: OrderLines[], shippingInfo: ShippingInfo) {
    this.shippingInfo = shippingInfo;
    this.orderLines = orderLines;
    this.totalAmount = new Money(this.calculateTotalAmount());
  }

  static From(orderLines: OrderLines[]) {
    if (this.orderLines.length === 0) {
      throw new Error("No order lines");
    }

    if (!this.shippingInfo) {
      throw new Error("No shipping info");
    }

    return new Order(orderLines);
  }

  private calculateTotalAmount() {
    return this.orderLines.reduce(
      (total, orderLine) => total + orderLine.getTotalPrice(),
      0
    );
  }

  changeShippingInfo(shippingInfo: ShippingInfo) {
    if (!this.isShippingChangeable()) {
      throw new Error(`Can't change shipping info in ${this.orderStatus}`);
    }
    this.shippingInfo = shippingInfo;
  }

  cancel() {
    if (this.orderStatus >= OrderStatus.SHIPPED) {
      throw new Error(`Can't cancel after shipped`);
    }
    this.orderStatus = OrderStatus.CANCELED;
  }

  private isShippingChangeable(): boolean {
    return (
      this.orderStatus === OrderStatus.PAYMENT_WAITING ||
      this.orderStatus === OrderStatus.PREPARING
    );
  }
}

class OrderLine {
  private product: Product;
  private price: Money;
  private quantity: number;

  constructor(product: Product, price: Money, quantity: number) {
    this.product = product;
    this.price = price;
    this.quantity = quantity;
  }

  getTotalPrice() {
    return this.price.amount * this.quantity;
  }
}
```

- 이런식으로 구현될 수 있다.
- 모델은 문서화하여 관리하는 것이 좋다.

## 엔티티와 벨류

### 엔티티

- 식별자를 가짐.
  - 주문의 경우 주문번호

### 밸류

- 개념적으로 완전한 하나

```ts
class ShippingInfo {
  private receiverName: string;
  private receiverPhoneNumber: string;
  private shippingAddress1: string;
  private shippingAddress2: string;
  private shippingZipcode: string;

  ...
}
```

여기서 `receiverName` `receiverPhoneNumber`은 받는 사람이라는 하나의 개념이 될 수 있으며
주소 정보 또한 하나의 개념이 될 수 있다.

```ts
//value-object.ts
class Receiver {
  name: string;
  phoneNumber: string;

  constructor(name: string, phoneNumber: string) {
    this.name = name;
    this.phoneNumber = phoneNumber;
  }
}

class Address {
  address1: string;
  address2: string;
  zipcode: string;

  constructor(address1: string, address2: string, zipcode: string) {
    this.address1 = address1;
    this.address2 = address2;
    this.zipcode = zipcode;
  }
}
```

이렇게 하면 `ShippingInfo`는 다음과 같이 변경할 수 있다.

```ts
class ShippingInfo {
  private receiver: Receiver;
  private address: Address;

  constructor(receiver: Receiver, address: Address) {
    this.receiver = receiver;
    this.address = address;
  }
}
```

Value는 항상 두개 이상의 데이터를 가져야만 하는 것은 아니다.

```ts
class Money {
  value: number;

  constructor(value: number) {
    this.value = value;
  }

  add(amount: Money) {
    return new Money(this.value + amount.value);
  }

  multiply(multiplier: number) {
    return new Money(this.value * multiplier);
  }
}
```

이렇게하면 Money는 금액이라는 의미를 명확하게 표현할 수 있으며 multiply는 돈계산이라는 의미가 된다.
또한 새로운 클래스로 만들어서 반환하는데 이를 불변이라고 표현한다. 이렇게하면 안전한 코드를 작성할 수 있다.

### 엔티티에 무분별한 setter 넣지 않기

이건 타입스크립트에서는 필요 없지 않나 싶다. setter말고 조금 더 범용적인 update 메서드를 사용할 수 있지 않을까 생각된다.

```ts
class Person {
  private name: string;
  private age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  update(args:UpdateArgs){
    const changedArgs = stripUnchanged(args)
    if(!changedArgs){
      return
    }
    this.validate()
    Object.assign(this, changedArgs)
  }

  private validate(){
    ...
  }
}
```

## 도메인 용어와 유비쿼터스 용어

코드에서 최대한 도메인용어를 쓰게되면 개발자와 기획자, 온라인 쇼핑 도메인 전문가가 이야기할 때 용어를 서로 해석해야되는 일이 줄어든다.
이는 생산성을 높인다.
