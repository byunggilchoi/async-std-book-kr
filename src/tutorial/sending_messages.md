## 메시지 보내기

이제 나머지 절반인 메시지 전송을 구현할 차례입니다.
전송을 구현하는 가장 확실한 방법은 각 `connection_loop`에 서로의 클라이언트 `TcpStream`에 대한 쓰기 권한을 주는 것입니다.
이렇게 하면 클라이언트가 수신자에게 직접 메시지를 `.write_all`을 이용해서 기록할 수 있게 됩니다.
하지만 이것은 잘못된 방법입니다. 만약 Alice가 `bob: foo`라고 보내고 Charley가 `bob: bar`라고 보냈을 때 Bob은 `fobaor`를 받을 수도 있기 때문입니다.(역자 주: 작성이 동시에 이뤄져서 말들이 섞일(`foo`, `bar` -> `fo`+`ba`+`o`+`r`) 수 있다는 뜻입니다. 하나의 쓰기가 완료되고 그 다음에 써야 하는 거죠.)
소켓을 통해서 메시지를 보내는 것은 여러 시스템 호출이 필요할 수도 있습니다. 그래서 `.write_all` 두 개를 동시에 호출하면 서로 간섭이 일어날 수도 있습니다.

가장 중요한 원칙은 하나의 Task만 각 `TcpStream`에 작성해야 한다는 것입니다.
그러면 채널을 통해 메시지를 받아서 소켓을 통해 메시지를 기록하는 `connection_writer_loop` Task를 만들어 봅시다.
이 Task는 메시지 직렬화 지점에 해당합니다.
Alice와 Charley가 Bob에게 동시에 두 메시지를 보냈을 때 Bob은 채널에 도착한 것과 동일한 순서로 메시지를 보게 됩니다.

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::{
#     net::TcpStream,
#     prelude::*,
# };
use futures::channel::mpsc; // 1
use futures::sink::SinkExt;
use std::sync::Arc;

# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
type Sender<T> = mpsc::UnboundedSender<T>; // 2
type Receiver<T> = mpsc::UnboundedReceiver<T>;

async fn connection_writer_loop(
    mut messages: Receiver<String>,
    stream: Arc<TcpStream>, // 3
) -> Result<()> {
    let mut stream = &*stream;
    while let Some(msg) = messages.next().await {
        stream.write_all(msg.as_bytes()).await?;
    }
    Ok(())
}
```

1. `futures` 크레잇에서 channel을 사용합니다.
2. 단순하게 하기 위해 이번 튜토리얼에서는 `unbounded` 채널을 사용하고 백프레셔에 대해서는 다루지 않겠습니다.
3. `connection_loop`와 `connection_writer_loop`는 똑같은 `TcpStream`을 공유합니다. 그 TcpStream을 `Arc`에 넣었습니다.
   `client`는 스트림에서 읽기만 하고 `connection_writer_loop`는 스트림에서 쓰기만 하기 때문에 서로 다툴 일이 없다는 점을 주목하세요.
