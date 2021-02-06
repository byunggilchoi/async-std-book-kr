# 요구사항과 작업 시작

## 요구사항

채팅 프로그램은 TCP를 이용한 단순한 텍스트 프로토콜을 이용합니다.
프로토콜은 utf-8 메시지로 구성되고 `\n`으로 구분합니다.

클라이언트는 서버에 접속해서 첫 줄에 로그인 정보를 보냅니다.
그리고 클라이언트는 다음 문법에 맞춰서 다른 클라이언트들에게 메시지를 보냅니다.

```text
login1, login2, ... loginN: message
```

개별 클라이언트들은 `from login: message`라는 메시지를 받습니다.

아마도 세션은 이런 식으로 이뤄지겠죠.

```text
On Alice's computer:   |   On Bob's computer:

> alice                |   > bob
> bob: hello               < from alice: hello
                       |   > alice, bob: hi!
                           < from bob: hi!
< from bob: hi!        |
```

채팅 서버의 가장 큰 문제는 수많은 동시 접속을 추적하는 것입니다.
채팅 클라이언트의 가장 큰 문제는 동시에 일어나는 메시지 수신, 발신 그리고 사용자의 타이핑까지 관리하는 것입니다.

## 작업 시작

먼저 Cargo 프로젝트를 만들어봅시다.

```bash
$ cargo new a-chat
$ cd a-chat
```

그리고 아래의 내용을 `Cargo.toml`에 추가하세요.

```toml
[dependencies]
futures = "0.3.0"
async-std = "1"
```
