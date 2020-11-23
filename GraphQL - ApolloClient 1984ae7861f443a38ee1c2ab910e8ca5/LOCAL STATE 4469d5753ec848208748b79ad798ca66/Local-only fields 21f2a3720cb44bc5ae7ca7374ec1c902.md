# Local-only fields

---

> Apollo Client 쿼리에는 GraphQL 서버의 스키마에 정의 되지 않은 **로컬 전용 필드**가 포함될 수 있다.

이러한 필드의 값은 데이터 읽기와 같이 사용자가 원하는 로직을 통해 계산된다. Ex) `**localStorage**`

## 예시로 알아보는 정의

예를들어 전자 상거래 App을 구축하고 있다고 가정해보자.

대부분의 제품 세부 정보는 백엔드 서버에 저장되지만, 클라이언트에서 독단적으로 관리할 boolean 필드 (=`**isInCart**`)가 필요할 경우가 있다.

이 필드를 정의하려면 우선, `**isInCart`** 에 해당하는  `**field policy**` (=필드 정책)을 만들어야 한다.

필드 정책은 단일 GraphQL 필드를 Apollo Client 캐시에서 가져오고 쓰는 방법에 대해 개발자가 커스터마이징 하는 부분이다.

로컬전용 필드(client 단)와 원격으로 가져온 필드(server 단) 모두에 대한 필드 정책을 정의할 수 있다.

`InMemoryCache` 를 사용하여 App 의 필드 정책을 정의.

```jsx
const cache = new InMemoryCache({
  typePolicies: { // 정책 필드의 Type
    Product: {
      fields: { // Product 타입의 정책필드 선언
        isInCart: { // Product 타입안에 Client 단만의 정책 필드 정의
          read(_, { variables }) { // The read function for the isInCart field
            return localStorage.getItem('CART').includes(
              variables.productId
            );
          }
        }
      }
    }
  }
})
```

위의 필드 정책은 필드에 대한 `**read`** 기능을 `**isInCart`**  에서 정의하여 사용하고있다.

`**read`** 함수가 있는 필드를 쿼리 할 때마다 캐시는 해당 함수를 호출하여 필드 값을 계산한다.

예시의 경우 `**read`** 함수는 쿼리 된 제품의 ID가 로컬스토리지의 `CART` 배열에 있는지 여부를 반환한다.

`**read**` 기능을 사용하면 ?

1. 캐시 작업을 수동으로 진행
2. 다른 헬퍼 유틸리티 또는 라이브러리를 호출하여 데이터 준비, 유효성 검사 또는 삭제 ( 프론트 로직 커스터마이징 )
3. 별도의 저장소에서 다른 데이터 가져오기
4. 사용량 메트릭 로깅 ( 사용량 체크 )

## Querying

`**isInCart`** 라는 정책 필드를 정의했으면, 백엔드에서 데이터를 받을 쿼리문을 작성할 대 아래와 같이 사용할 수 있다.

```jsx
const GET_PRODUCT_DETAILS = gql`
  query ProductDetails($productId: ID!) {
    product(id: $productId) {
      name
      price
      isInCart @client
    }
  }
`;
```

`@clien` 는  Apollo Client 에게 `**isInCart`** 가  로컬 전용 필드임을 알려주는 구문이다.

따라서 백엔드는 위에 정의한 name, price 만 반환하고, 반환 후 `**isInCart`** 를 실행하여 최종 쿼리 결과에 반환하는 형식으로 진행된다.

## Storing (저장)

기존 Apollo Client 를 사용하여 상태 저장하는 방법과는 관계없이 로컬 상태를 쿼리할 수 있다.

Apollo Client 는 로컬 상태를 표시하기 위한 몇가지 선택사항으로 유용한 기능을 제공한다,

1. Reactive variables (반응 변수)
2. Apollo Client 캐시

### Storing local state in reactive variables

(반응 변수에 로컬 상태 저장)

1. GraphQL 로직을 사용하지 않고도 App의 어느 곳에서다 반응 변수를 읽고 수정할 수 있다.
2. Apollo Client 캐시와 달리, 반응 변수는 엄격한 규칙을 따르지 않으므로 원하는 형식으로 데이터를 저장할 수 있다.
3. 필드의 값이 반응 변수에 값에 따라 달라지고, 해당 변수의 값이 변경되면 **해당 필드를 포함하는 모든 활성 쿼리가 자동으로 새로고침된다. ? ( 활성 쿼리 공부하자 )**

- 예시로 알아보는 정의

IF ) 사용자의 장바구니에 있는 항목 ID 를 가져오려고 한다. ID 목록은 로컬에 저장된다.

쿼리

```jsx
// Cart.js
export const GET_CART_ITEMS = gql`
  query GetCartItems {
    cartItems @client
  }
`
```

다음과 같이 장바구니 항목의 로컬 목록을 저장하기 위해 반응 변수를 초기화 하자.

```jsx
// cache.js
export const cartItemsVar = makeVar([])
```

빈 배열을 포함하는 반응 변수를 초기화.

배열안에 값을 넣고 싶다면 **`cartItemsVar (newValue)`** 를 정의하여 새 값을 설정할 수 있다.

필드 정책 정의 ➡ `InMemoryCache`

```jsx
// cache.js
export const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        cartItems: {
          read() {
            return cartItemsVar();
          }
        }
      }
    }
  }
})
```

`read` 함수의 `cartItems` 는 쿼리 될 때마다 반응 변수의 값을 반환한다.

이제, 사용자가 장바구니에 제품을 추가할 수 있는 버튼 구성요소를 만들어보자

```jsx
// AddToCartButton.js

import { cartItemsVar } from './cache';

export function AddToCartButton({ productId }) {
  return (
    <div class="add-to-cart-button">
      <Button onClick={() => cartItemsVar([...cartItemsVar(), productId])}>
        Add to Cart
      </Button>
    </div>
  );
}
```

onClick 이벤트로 처리된 cartItemsVar 는 Apollo Client 의 cartItems 필드를 포함한 모든 활성 쿼리에 알린다.

이후 `Cart` 는 `GET_CART_ITEMS` 쿼리를 사용함으로 `cartItemsVar` 로 인해 값이 변경 될 때마다 자동으로 새롭게 고쳐지는 구성 요소이다.

```jsx
// Cart.js

export const GET_CART_ITEMS = gql`
  query GetCartItems {
    cartItems **@client**
  }
`;

export function Cart() {
  const { data, loading, error } = useQuery(GET_CART_ITEMS);

  if (loading) return <Loading />;
  if (error) return <p>ERROR: {error.message}</p>;

  return (
    <div class="cart">
      <Header>My Cart</Header>
      {data && data.cartItems.length === 0 ? (
        <p>No items in your cart</p>
      ) : (
        <Fragment>
          {data && data.cartItems.map(productId => (
            <CartItem key={productId} />
          ))}
        </Fragment>
      )}
    </div>
  );
}
```

useQuery를 사용하여 업데이트 된 값을 받아오는 케이스도 있고, Apollo Client 3.2.0 버젼부터 새롭게 도입된

`useReactiveVar` Hook를 통해 반응 변수에서 직접 읽을 수 있다.

```jsx
// Cart.js

import { useReactiveVar } from '@apollo/client';

export function Cart() {
  const cartItems = useReactiveVar(cartItemsVar);

  return (
    <div class="cart">
      <Header>My Cart</Header>
      {cartItems.length === 0 ? (
        <p>No items in your cart</p>
      ) : (
        <Fragment>
          {cartItems.map(productId => (
            <CartItem key={productId} />
          ))}
        </Fragment>
      )}
    </div>
  );
}
```

`useReactiveVar`  가 기존 useQuery 를 사용하여 불러오는 방법과 완전히 같은것은 아니다.

다른점이라면, `useReactiveVar` 를 사용하여 데이터를 불러오면 @client 를 통해 처리되는 방식이 아니므로 향후 변수 업데이트는 구성 요소를 **다시 렌더링 하지 않는 다는 점**이 있다 ( 한마디로 일회성 )

### Storing local state in the cache

( 캐시에 로컬 상태 저장 )

로컬 상태를 Apollo Client 캐시에 직접 저장하면 몇 가지 장점이 있지만, 일반적으로 반응 변수를 사용하는 것 보다 더 많은 코드가 필요하다.

1. read 함수가 없는 일반적인 필드를 쿼리하는 경우 캐싱처리는 Apollo Client 에서 직접 해당 필드의 값을 가져오는 식으로 캐싱이 된다.
2. 캐시 필드를 수정하는 경우 `writeQuery` 나 `writeFragment` 를 사용하여 자동으로 필드를 새로고침할 수 있는 활성쿼리를 작성한다.

```jsx
const IS_LOGGED_IN = gql`
  query IsUserLoggedIn {
    isLoggedIn @client
  }
`;
```

`isLoggedIn` 는 로컬 전용 필드이다. 이 쿼리에 대해 `writeQuery` 메서드를 사용하여 이 필드의 캐시에 직접적으로 값을 작성해야한다

```jsx
cache.writeQuery({
  query: IS_LOGGED_IN,
  data: {
    isLoggedIn: !!localStorage.getItem("token"),
  },
});
```

localStorage 의 값을 확인하여 isLoggedIn의 값을 동적으로 바뀌는 형식의 필드를 구성.

이렇게 작성된 쿼리는

메인페이지에 위치하여 사용자가 로그인 유무에 따라 작동하는 쿼리구문으로 동작할 수 있다.

```jsx
function App() {
  const { data } = useQuery(IS_LOGGED_IN);
  return data.isLoggedIn ? <Pages /> : <Login />;
}
```

## Modifying (수정)

로컬 전용 필드의 값을 수정하는 방법은 해당 필드를 저장하는 방법에 따라 다르다.