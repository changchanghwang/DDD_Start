# 애그리거트 트랜젝션 관리

운영자가 주문 상태를 배송중으로 변경하려고 할 때 주문자가 배송지 변경을 하게된다면 충돌이 생길 수 있기 때문에 트랜잭션이 필요하다.

## Pessimistic Lock

Pessimistic Lock(책에서는 이해하기 쉽도록 선점 잠금이라는 용어를 사용했다.)은
데이터를 사용하는 시점에 데이터를 잠그는 방식이다. 그래서 다른 사용자는 해당 애그리거트가 잠금이 해제될 때까지 기다려야 한다.

- 운영자가 먼저 주문 애그리거트를 변경하려고 하면 고객은 변경이 완료될때 까지는 조회가 불가능하다.
- 조회를 시작하는 시점에서 운영자의 변경이 끝날때 까지는 블로킹 되어있다가 변경이 끝나면 조회를 하게되고,
- 그러면 배송 상태이기 때문에 배송지 변경에 실패하게 된다.

### 주의점

DeadLock이 걸리지 않도록 조심한다.

A -> B
B -> A
일때 서로 락이 걸려서 교착상태가 될 수 있다. 이런 상황을 Dead Lock이라고 한다.
그래서 Lock은 최대 대기 시간을 지정하는게 좋으며 최대한 이런 Dead Lock이 생기지않게 해야한다.

## Optimistic Lock

Optimistic Lock (책에서는 이해하기 쉽도록 비선점 잠금이라고 하고있다.)은 데이터 접근은 허용하지만 값을 수정한 것을 명시하여 다른 사용자가 같은 조건에서 동시에 수정하는 것을 방지하는 방식이다.

이를 구현하려면 애그리거트에 버전으로 사용할 프로퍼티를 추가하고 수정될 때마다 버전을 1씩 증가하면 된다.

- 운영자가 주문 애그리거트를 변경하는 도중에 (주문 version: 1)
- 주문자가 배송지변경을 위해 조회한다. (주문 version: 1)
- 운영자가 주문 애그리거트를 변경하고 저장한다. (주문 version: 2)
- 주문자가 배송지 변경한다. -> 이때 주문자는 배송지 변경에 실패한다. (주문 version 1의 주문이 2로 변경되었기 때문에)

### 애그리거트 루트 버전 증가

애그리거트 루트 외에 다른 앤티티의 값만 변경한다고 했을 때 루트 앤티티가 변경이 되지 않는다면 문제가 생길 수 있다.
애그리거트 관점에서 보면 문제가 될 수 있기 때문에 루트 애그리거트의 버전을 증가하도록 한다.

### Offline Pessimistic Lock

단일 트랜잭션에서 동시 변경을 막는 Pessimistic Lock과는 달리 여러 트랜잭션에 걸쳐 동시 변경을 막는다. 첫번째 트랜잭션에서 잠금을 걸고 마지막 트랜잭션에서 잠금을 푼다.
보통 수정 기능은 2개의 트랜잭션으로 구성된다.

1. 조회
2. 수정

오프라인 선점 잠금은 수정 요청을 하지 않고 프로그램을 종료하면 잠금을 해제하지 않아 다른 사용자는 영원히 잠금을 구할 수 없게 된다.
그래서 잠금 유효 시간을 두어야한다.
-> 하지만 잠금 유효 시간을 그대로 놔둔다면 수정이 오래걸리는 작업에서는 수정에 실패하기 때문에 일정 주기로 유효시간을 연장해주는 방법이 필요하다.

주로 WIKI에서 수정폼을 수정할 때 사용할 수 있다.

```ts
class LockManager {
  tryLock(type: string, id: string): LockId;
  checkLock(lockId: LockId): void;
  releaseLock(lockId: LockId): void;
  extendLockExpiration(lockId: LockId, time: number): void;
}
```

tryLock의 경우 type으로 잠굴 entity, id로 entity id를 받는다.
LockId의 경우 추후에 잠금 해제, 잠금 확인등을 위해 필요하다. 그래서 어딘가에 저장해 놓아야 하기 때문에 모델에 추가한다.

```ts
class LockId {
  value: string;

  constructor(value: string) {
    this.value = value;
  }
}
```

```ts
class DataService {
  ...
  getDataWithLock(id: string) {
    //Lock 시도
    const lockId = lockManager.tryLock("data", id);

    // 기능 실행
    const data = dataRepository.findById(id);
    return {
      ...data,
      lockId,
    };
  }
}
```

잠금을 선점하는 데 실패하면 에러를 발생시키고 다른 사용자가 데이터를 수정 중이니 나중에 다시 시도하라는 메세지를 보내주면 된다.
