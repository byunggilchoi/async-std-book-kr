## 수락 루프 작성

서버의 얼개부터 구현해보겠습니다. TCP 소켓을 주소에 바인딩하고 연결 수락을 시작하는 루프죠.

먼저 사용할 라이브러리 요소들부터 추가해봅시다.

```rust,edition2018
# extern crate async_std;
use async_std::{
    prelude::*, // 1
    task, // 2
    net::{TcpListener, ToSocketAddrs}, // 3
};

type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>; // 4
```

1. `prelude`는 future와 stream으로 작업하기 위해 필요한 트레잇들을 담고 있습니다.
2. `task` 모듈은 대략 `std::thread` 모듈과 비슷하지만 task 모듈이 훨씬 더 가볍습니다.
   단일 쓰레드는 여러 개의 task를 처리할 수 있습니다.
3. 소켓 타입에서는 `async_std`의 `TcpListener`를 사용할 것입니다. 이는 `std::net::TcpListener`와 비슷하지만 논블로킹이고 `async` API를 사용합니다.
4. 이번 예제에서는 포괄적인 에러 처리는 구현하지 않을 것입니다.
   에러를 전파하기 위해서 boxed 에러 트레잇 객체를 사용합니다.
   std 라이브러리에 `From<&'_ str> for Box<dyn Error>` 구현이 있다는 것을 알고 있었나요? `?` 연산자를 문자열과 함께 사용할 수 있습니다.

이제 서버의 수락 루프를 작성할 수 있게 되었습니다.

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     net::{TcpListener, ToSocketAddrs},
#     prelude::*,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> { // 1

    let listener = TcpListener::bind(addr).await?; // 2
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await { // 3
        // TODO
    }
    Ok(())
}
```

1. `accept_loop` 함수에 `async`를 표시하면 함수 안에서 `.await` 문법을 쓸 수 있습니다.
2. `TcpListener::bind` 호출은 Future를 반환합니다. 여기에 `.await`를 사용해서 `Result`를 받을 수 있고 `?` 연산자를 적용하면 `TcpListener`가 나옵니다.
   어떻게 `.await`와 `?` 연산자가 어떻게 함께 잘 작동하는지 꼭 확인합시다.
   정확하게 `std::net::TcpListener`가 작동하는 방법인데 여기에 `.await`만 추가되었습니다.
   `std` API를 그대로 복제하는 것이 `async_std`의 명확한 디자인 목표이기 때문입니다.
3. 이제 우리는 소켓으로 들어오는 값들을 반복하고 싶습니다. `std`에서는 아래와 같이 작동하는 것처럼요.

```rust,edition2018,should_panic
let listener: std::net::TcpListener = unimplemented!();
for stream in listener.incoming() {
}
```

하지만 안타깝게도 이 기능은 `async`에서 아직 작동하지 않습니다. Rust에서 `async` for-반복문을 지원하지 않기 때문입니다.
따라서 `while let Some(item) = iter.next().await` 패턴을 사용해서 아래와 같이 루프를 수동으로 구현해야 합니다.

마지막으로 main을 추가하겠습니다.

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     net::{TcpListener, ToSocketAddrs},
#     prelude::*,
#     task,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
# async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> { // 1
#     let listener = TcpListener::bind(addr).await?; // 2
#     let mut incoming = listener.incoming();
#     while let Some(stream) = incoming.next().await { // 3
#         // TODO
#     }
#     Ok(())
# }
#
// main
fn run() -> Result<()> {
    let fut = accept_loop("127.0.0.1:8080");
    task::block_on(fut)
}
```

여기서 깨달아야 할 중요한 점은 다른 언어들과는 다르게 Rust에서는 비동기 함수를 호출해도 실행이 되지 **않는다는** 점입니다.
비동기 함수는 비활성 상태 머신인 Future를 만들 뿐입니다.
비동기 함수에서 Future 상태 머신을 단계 별로 실행하려면 `.await`를 사용해야 합니다.
비동기가 아닌 함수에서는 Future를 실행하려면 이를 실행자(executor)에게 전달해야 합니다.
이 경우에는 `task::block_on`을 이용해서 Future를 현재의 쓰레드에서 실행하고 완료될 때까지 나머지 작업을 블로킹하게 됩니다.
