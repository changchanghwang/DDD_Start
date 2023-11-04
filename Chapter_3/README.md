# 애그리거트

상위 수준의 개념을 이용해서 모델을 정리하면 전반적인 관계를 파악하기 쉽다.
애그리거트는 개념적으로 관련된 객체들을 하나의 군으로 묶은 군이기 때문에 상위수준으로 본다면 관계파악이 쉬워진다.

하나의 애그리거트에 속한 객체들은 거의 동일한 라이프 사이클을 갖는다.
애그리거트 단위로 일관성을 관리하게 되면 도메인을 보다 단순한 구조로 만들어준다. -> 생산성 증가

`A가 B를 갖는다` 일때 A와 B를 한 애그리거트로 묶어서 생각하기 쉽지만 꼭 그렇지만은 않다.
예를들어 상품과 리뷰는 라이프사이클이 서로 다르고 사용자도 다르기때문에 다른 애그리거트다.

## 애그리거트 루트

- Order
- OrderLine

위와 같이 두 객체가 있다고 가정했을 때 OrderLine의 개수를 변경하게 된다면 Order의 총 금액도 변경되어야한다.
그렇지 않으면 도메인 규칙도 어기게 되고 데이터 일관성이 깨지게된다.
그래서 일관된 상태를 유지하기위해서 애그리거트 전체를 관리할 주체가 필요하고 이 책임을 지는 것이 애그리거트의 루트 엔티티다.

```ts
class Order {
    ...
    changeShippingInfo(shippingInfo: ShippingInfo) {
        this.shippingInfo = shippingInfo;
    }
}
```

이렇게 order에서 shippingInfo를 변경하도록 하고 `shippingInfo.setAddress`와 같이 직접 변경하게되면 안된다.

## 트랜젝션 범위

작을수록 좋다. Lock의 대상이 적을수록 성능이 좋기때문이다. Lock 대상이 많아지면 그만큼 동시에 처리할 수 있는 트랜젝션의 개수가 줄어들기 때문에 처리량이 줄어든다.
그래서 한 트랙젝션에서는 한 개의 애그리거트만 수정해야 한다.

## 애그리거트를 팩토리로 사용하기

```ts
class RegisterProductService {
  registerNewProduct(args: CreateProductDto) {
    const store = await this.storeRepository.findById(args.storeId);
    if (store.isBlocked()) {
      throw new Error("Store is blocked");
    }

    const product = new Product(args.productId, ...args);
    await this.productRepository.save(product);
  }
}
```

위와 같이 코드를 했었을때 `스토어가 block되면 새로운 상품을 등록하지 못한다` 라는 비즈니스 로직이 application layer에 노출되어있다.

```ts
class Store {
  createProduct(productId: string, args: ProductCtorType) {
    if (isBlocked()) throw new Error("blocked");
    return new Product(productId, ...args);
  }
}
```

위와같이 Store 애그리거트의 createProduct는 Product 애그리거트를 생성하는 팩토리 역할을 하도록 하고 비즈니스 로직을 넣게 되면
도메인 로직을 구현하고 있기 때문에 이제 application service에서는 다음과 같이 코드를 짜면 된다

```ts
class RegisterProductService {
  registerNewProduct(args: CreateProductDto) {
    const store = await this.storeRepository.findById(args.storeId);
    const product = store.createProduct(args.productId, ...args);
    await this.productRepository.save(product);
  }
}
```
