# 도메인 서비스

## 여러 애그리거트가 필요한 기능

결제 금액을 계산할 때 다음과 같은 애그리거트들이 필요하다

- 상품: 상품별 가격, 상품에 따른 배송비가 상이할 수 있음.
- 주문: 상품별 구매 개수
- 할인 쿠폰: 할인 금액, 비율에 따른 주문 총 금액 할인. 조건에 따라 중복 사용할 수 있다거나 지정한 카테고리의 삼품에서만 적용할 수 있다면 더 복잡해진다.
- 회원: 회원 등급에 따른 할인. 회원 등급에 따라 배송비를 무료로 할 수도 있다.

이상황에서 주체가 되는건 어떤 애그리거트인가가 가장 복잡한 것 같다.
종 주문 금액을 계산하는 건 주문 애그리거트의 책임이지만 총 결제 금액은 주문 금액과 다르다.

그렇다면 주문 애그리거트가 필요한 데이터를 모두 가지도록 한 뒤 할인 금액 계산 책임을 주문 애그리거트에 할당한다면?

```ts
class Order {
  private orderer: Member;
  private oderLines: OrderLine[];
  private usedCoupons: Coupon[];

  private calculatePayAmounts() {
    const totalAmounts = this.calculateTotalAmounts();
    // 쿠폰 별 할인 금액 계산
    const discountRates = coupons
      .map((coupon) => this.calculateDiscount(coupon))
      .reduce((acc, cur) => acc + cur, 0);

    // 회원 등급에 따른 할인 금액 계산
    const membershipDiscountRates = this.calculateMembershipDiscountRates(
      this.orderer
    );

    return totalAmounts - discountRates - membershipDiscountRates;
  }

  private calculateDiscount(coupon: Coupon) {
    // 각 상품에 대해 쿠폰을 적용해서 할인 금액을 계산하는 로직
    // 쿠폰의 적용 조건 등을 확인하는 로직
  }

  private calculateMembershipDiscountRates(orderer: Member) {
    // 회원 등급에 따른 할인 금액 계산
  }
}
```

이렇게 하면 주문 애그리거트가 결제 금액 계산 로직의 책임을 가진다. 하지만 정말 이것이 옳은 것이라고 한다면 조금 이상할 수 있다.
예를 들어 특별 감사 세일을 하기로 했다고 가정해보면 할인 정책이 주문 애그리거트가 갖고 있는 구성요소와는 관련이 없음에도 주문 애그리거트를 수정해야한다.

이렇게 책임이 애매한 도메인 기능은 특정 애그리거트에 구현하게 되면 자시느이 책임을 넘는 기능을 구현하기 때문에 외부 의존도가 높아지며 코드가 복잡하게 될 수 있다.

## 도메인 서비스

도메인 서비스는 도메인의 의미가 드러나는 이름을 갖는다.

```ts
class CalculateDiscountedRateService {
  calculate(orderLines: OrderLine[], coupons: Coupon[], orderer: Member) {
    const couponDiscountRates = coupons
      .map((coupon) => this.calculateCouponDiscountRate(coupon))
      .reduce((acc, cur) => acc + cur, 0);
    const member;
    const membershipDiscountRates = this.calculateMembershipDiscountRates(
      this.orderer
    );

    return totalAmounts - discountRates - membershipDiscountRates;
  }

  private calculateCouponDiscountRate(coupon: Coupon) {
    // 각 상품에 대해 쿠폰을 적용해서 할인 금액을 계산하는 로직
    // 쿠폰의 적용 조건 등을 확인하는 로직
  }

  private calculateMembershipDiscountRates(orderer: Member) {
    // 회원 등급에 따른 할인 금액 계산
  }
}
```

이 도메인 서비스를 사용하는 주체는 애그리거트가 될 수도 있고 응용 서비스가 될 수도 있다.
위에서는 Order에서 로직을 구현했었기때문에 애그리거트에 넣을 수 잇다.

```ts
class Order {
  calculateAmounts(
    calculateDiscountedRateService: CalculateDiscountedRateService
  ) {
    const totalAmounts = this.getTotalAmounts();
    const discountedRates = calculateDiscountedRateService.calculate(
      this.orderLines,
      this.coupons,
      this.orderer
    );

    this.paidAmounts = totalAmounts - discountedRates;
  }
}
```

그리고 애그리거트에 도메인 서비스를 전달하는 것은 응용 서비스의 책임이다.

```ts
class OrderService {
  private calculateDiscountedRateService: CalculateDiscountedRateService;
  ...

  order(orderRequest:OrderRequest){
    ...
    const order = new Order(this.calculateDiscountedRateService);

    this.orderRepository.save(order);
    return order;
  }
}
```

### 도메인 서비스 객체를 애그리거트에 직접 주입하지 않는다.

도메인 서비스 객체를 애그리거트에 직접 주입하게 되면 해당 애그리거트는 도메인 서비스에 의존한다는 것을 의미한다.
하지만 해당 도메인 서비스는 애그리거트의 데이터 자체와는 관련이 없기때문에 따로 저장할 필요가 없다.

애그리거트 메서드를 실행할 때 도메인 서비스를 인자로 전달하지 않고 도메인 서비스의 기능을 실행할 때 애그리거트를 전달하기도 한다.
계좌 이체 기능이 그 예다.

```ts
class TransferService {
  transfer(from: Account, to: Account, amounts) {
    from.withDraw(amounts);
    to.deposit(amounts);
  }
}
```

도메인 서비스는 도메인 로직을 수행하지 응용 로직을 수행하진 않기때문에 트랜잭션 같은 응용 로직은 응용 서비스 레벨에서 처리한다.
