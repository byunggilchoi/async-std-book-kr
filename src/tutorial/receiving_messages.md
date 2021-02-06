## 메시지 받기

프로토콜에서 수신부를 구현해봅시다.
아래 세 가지를 해야 합니다.

1. 받은 `TcpStream`을 `\n` 단위로 분리해서 utf-8 단위의 바이트로 디코딩
2. 첫번째 줄은 login으로 처리
3. 나머지는 `login: message`으로 파싱

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
use async_std::{
    io::BufReader,
    net::TcpStream,
};

async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await {
        let stream = stream?;
        println!("Accepting from: {}", stream.peer_addr()?);
        let _handle = task::spawn(connection_loop(stream)); // 1
    }
    Ok(())
}

async fn connection_loop(stream: TcpStream) -> Result<()> {
    let reader = BufReader::new(&stream); // 2
    let mut lines = reader.lines();

    let name = match lines.next().await { // 3
        None => Err("peer disconnected immediately")?,
        Some(line) => line?,
    };
    println!("name = {}", name);

    while let Some(line) = lines.next().await { // 4
        let line = line?;
        let (dest, msg) = match line.find(':') { // 5
            None => continue,
            Some(idx) => (&line[..idx], line[idx + 1 ..].trim()),
        };
        let dest: Vec<String> = dest.split(',').map(|name| name.trim().to_string()).collect();
        let msg: String = msg.to_string();
    }
    Ok(())
}
```

1. `task::spawn` 함수를 이용해서 각 클라이언트와 작업하기 위한 독립적인 Task를 생성합니다.
   즉, 클라이언트를 수락하면 즉시 `accept_loop`이 다음 클라이언트를 기다리기 시작하는 것입니다.
   이벤트 주도 아키텍처의 핵심 이점입니다. 동시에 많은 클라이언트에 서비스를 제공하지만 하드웨어 쓰레드를 많이 쓰지는 않죠.

2. 운 좋게도 "바이트 스트림을 행으로 분할"하는 기능은 이미 구현이 되어 있습니다.
   `.lines()`를 호출하면 `String`의 스트림을 반환합니다.

3. 이제 첫번째 줄에서 로그인을 얻을 수 있습니다.

4. 그리고 다시 한 번, 수동 async 루프를 구현합니다.(역자 주: 아직 Rust에서 async의 for 루프를 지원하지 않습니다.)

5. 마지막으로 각 행을 대상 로그인 목록과 메시지 자체로 파싱합니다.

## 에러 관리

위의 솔루션이 가지는 중요한 문제는 `connection_loop`에서 오류를 올바르게 전파하지만 마지막에는 그냥 던져버린다는 것입니다.
`task::spawn`이 나왔다고 즉시 오류를 반환하지 않습니다.(사실 그렇게 할 수가 없습니다. Future의 작업이 완료되는 것이 먼저 필요하니까요.)
Task가 결합될 때까지 기다리도록 다음과 같이 수정할 수 있습니다.

```rust,edition2018
# #![feature(async_closure)]
# extern crate async_std;
# use async_std::{
#     io::BufReader,
#     net::{TcpListener, TcpStream, ToSocketAddrs},
#     prelude::*,
#     task,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
# async fn connection_loop(stream: TcpStream) -> Result<()> {
#     let reader = BufReader::new(&stream); // 2
#     let mut lines = reader.lines();
#
#     let name = match lines.next().await { // 3
#         None => Err("peer disconnected immediately")?,
#         Some(line) => line?,
#     };
#     println!("name = {}", name);
#
#     while let Some(line) = lines.next().await { // 4
#         let line = line?;
#         let (dest, msg) = match line.find(':') { // 5
#             None => continue,
#             Some(idx) => (&line[..idx], line[idx + 1 ..].trim()),
#         };
#         let dest: Vec<String> = dest.split(',').map(|name| name.trim().to_string()).collect();
#         let msg: String = msg.trim().to_string();
#     }
#     Ok(())
# }
#
# async move |stream| {
let handle = task::spawn(connection_loop(stream));
handle.await?
# };
```

`.await`을 통해 클라이언트가 종료할 때까지 기다리고 `?` 연산자로 결과값을 전파합니다.

하지만 이 솔루션에도 여전히 두 가지 문제가 있습니다!
*첫 번째*는 클라이언트 작업이 완료될 때까지 기다리는 방식이라서 한 번에 클라이언트를 하나만 처리할 수 있다는 점입니다. 그러면 비동기의 목적과 완전히 어긋나죠.
*두 번째*는 만약 클라이언트에 IO 에러가 발생하면 서버가 즉시 종료됩니다.
즉 하나의 피어만 연결이 불안정해도 모든 채팅방이 다운되는 거죠.

이 경우 클라이언트 오류를 처리하는 올바른 방법은 로그를 남기고 다른 클라이언트들에 서비스를 제공하는 것입니다.
이를 위해 헬퍼 함수를 사용해봅시다.

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     io,
#     prelude::*,
#     task,
# };
fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()>
where
    F: Future<Output = Result<()>> + Send + 'static,
{
    task::spawn(async move {
        if let Err(e) = fut.await {
            eprintln!("{}", e)
        }
    })
}
```
