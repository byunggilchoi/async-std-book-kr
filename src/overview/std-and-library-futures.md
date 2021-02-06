# `std::future`와 `futures-rs`

Rust에는 흔히 `Future`로 불리는 타입이 두 종류입니다.

- 첫번째는 `std::future::Future`로 Rust의 [기본 라이브러리](https://doc.rust-lang.org/std/future/trait.Future.html)에서 나온 거죠.
- 두번째는 `futures::future::Future`로 별도의 라이브러리인 [futures-rs 크레이트](https://docs.rs/futures/0.3/futures/prelude/trait.Future.html)에 있습니다.

[futures-rs](https://docs.rs/futures/0.3/futures/prelude/trait.Future.html) 크레이트에 정의되어 있는 future가 원래 있던 구현체입니다. `async/await` 문법을 사용하기 위해서 Future 트레잇 중 핵심 부분이 Rust의 기본 라이브러리로 옮겨 갔고`std::future::Future`가 되었습니다. 그러니까 어떻게 보면 `std::future::Future`는 `futures::future::Future`의 부분집합인 셈이죠.

`std::future::Future`와 `futures::future::Future`의 차이와 `async-std`가 사용하는 접근법을 이해하는 것은 중요합니다. Rust 사용자 입장에서 `std::future::Future` 자체를 상대하고 싶지는 않을 것입니다. `.await`을 호출하는 정도를 제외하면요. `std::future::Future` 내부에서 하는 작업들은 주로 `Future`를 구현하는 사람들에게나 관련이 있다고 보셔도 됩니다. 실수하지 않는 것이 굉장히 중요합니다. `Future` 자체에 정의되어 있던 기능들 대부분은 확장 트레잇인 [`FutureExt`](https://docs.rs/futures/0.3/futures/future/trait.FutureExt.html)로 옮겨갔습니다.. 즉 `futures` 라이브러리가 핵심 Rust 비동기 기능의 확장 역할을 한다는 것을 알 수 있습니다.

`futures`와 마찬가지로 `async-std`도 `std::future::Future` 타입을 그대로 쓸 수 있도록 포함하고 있습니다. `Cargo.toml`에 추가하고 `FutureExt`를 가져 오면 `futures` 크레이트에서 제공하는 확장 기능들을 선택하여 쓸 수 있습니다.

## 인터페이스와 안정성

`async-std`는 Rust 표준 라이브러리 수준으로 안정적이고 신뢰할 수 있는 라이브러리를 추구합니다. 즉 인터페이스 수준에서 `futures` 라이브러리에 의존하지 않습니다. 하지만 많은 사용자들이 `futures-rs`가 가져온 편리성을 좋아하게된 것에 대해서 고맙게 생각합니다. 그래서 `async-std`는 각 타입들에 대해서 `futures` 트레잇을 모두 구현해놓았습니다.

다행히 이런 접근 방식으로 인해 `futures`와 `async-std` 두 라이브러리의 관계를 완전히 유연하게 만들 수 있었습니다. 안정성에 관심이 많다면 `async-std`만 쓰면 됩니다. `futures` 라이브러리의 인터페이스를 선호한다면 두 가지를 연결해서 사용할 수도 있습니다. 두 가지 사용방식 모두 완전히 지원되니까요.

## `async_std::future`

`async_std::future` 모듈에서 여러 종류의 futures를 다루는데 중요하다고 여겨지는 함수들을 지원하고 있고 안정성도 보장합니다.

## Streams, Read/Write/Seek/BufRead 트레잇

Rust 컴파일러 상의 제약으로 `async_std` 내부에는 현재 구현이 되어 있지만 유저가 직접 구현할 수는 없습니다.
