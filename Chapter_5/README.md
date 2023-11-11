# 조회

## 검색을 위한 스펙

검색조건이 고정되어 있다면 특정 조건으로 조회하는 기능르 만들면 된다.
다만 목록 조회같은 기능은 다양한 검색 조건을 조합해야 할 때가 있다.
책과는 다르게 나는 이렇게구현할 수 있을 것 같다.

```ts
interface OrderSpec {
  findSatisfiedElements(orderRepository: OrderRepository): Order[];
}

class FilteredOrderSpec implements OrderSpec {
  private ids?: string;

  constructor(args: { ids?: string }) {
    this.ids = args.ids;
  }

  findSatisfiedElements(orderRepository: OrderRepository): Order[] {
    return orderRepository.find({ ids: this.ids });
  }
}
```

또한, 다양한 검색조건이 아닌 특정 검색조건을 만족하는 조회기능을 만들더라도 어떤 조건이 필요한지는 도메인의 영역이다. validation도 그래서 옮길 수 있다.

- 출고 전에 주문을 취소할 수 있다.

```ts
enum OrderStatus {
  PAYMENT_WAITING = 1,
  PREPARING = 2,
  SHIPPED = 3,
  DELIVERING = 4,
  DELIVERY_COMPLETED = 5,
  CANCELED = 6,
}

class CancelableOrderSpec implements OrderSpec {
  private id!: string;

  constructor(args: { id: string }) {
    this.id = args.id;
  }

  findSatisfiedElements(orderRepository: OrderRepository): Order[] {
    const order = orderRepository.findOneOrFail(this.id);

    if (this.orderStatus >= OrderStatus.SHIPPED) {
      throw new Error(`Can't cancel after shipped`);
    }

    return [order];
  }
}
```

이렇게 구현한다면 cancel 할 order를 찾을때부터 validation을 할 수 있다. 한군데 로직이 집중되어있기 때문에 유지보수가 쉽다.
