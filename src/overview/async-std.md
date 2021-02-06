# `async-std`에 오신 것을 환영합니다.

`async-std`는 [지원하는 다른 라이브러리들][organization]과 함게 비동기 프로그래밍 생활을 더 쉽게 만들어주는 라이브러리입니다. 다운스트림 라이브러리와 어플리케이션 모두에 대해서 기본적인 구현체들을 제공합니다. `async-std`라는 이름처럼 가능한 한 Rust 표준(`std`) 라이브러리에 가깝게 모델링되어 있고 모든 구성 요소를 비동기 방식의 요소로 대체하였습니다.

`async-std`는 파일 시스템, 네트워크 작업은 물론이고 타이머와 같은 기본적인 동시성까지 모든 중요한 기본 요소들에 대한 인터페이스를 제공합니다. 또한 Rust 표준 라이브러리에 있는 `thread` 모듈과 유사한 `task` 기능도 제공합니다. 게다가 I/O 기본 요소들 뿐만 아니라 `Mutex`(역자 주: 비동기에서 사용하는 자료 구조. rust book의 [해당 챕터][mutex] 참조.)의 `async/await` 버전에 해당하는 요소들도 있습니다.

[organization]: https://github.com/async-rs
[mutex]: https://doc.rust-lang.org/book/ch16-03-shared-state.html
