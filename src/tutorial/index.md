# 튜토리얼: 채팅 만들기

채팅 서버를 만드는 것보다 간단한 게 없죠. 안 그래요?
반드시 정답은 아니더라도 채팅서버를 통해 비동기 프로그래밍의 즐거움을 느낄 수 있을 것입니다.

어떻게 서버가 동시에 클라이언트들의 연결을 처리할까요?

어떻게 서버가 클라이언트들의 연결 해제를 처리할까요?

어떻게 서버가 메시지들을 뿌릴까요?

이번 튜토리얼은 `async-std`를 이용해서 어떻게 채팅 서버를 구현하는 지 설명할 것입니다.

[레포지터리](https://github.com/async-rs/async-std/blob/master/examples/a-chat)에서도 튜토리얼의 내용을 찾아볼 수 있습니다.