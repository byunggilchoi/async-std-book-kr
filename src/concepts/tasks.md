# Tasks

이제 Future가 뭔지 아니까 실행해봅시다.

`async-std`에서는 [`tasks`][tasks] 모듈이 Future의 실행을 담당합니다. 가장 간단한 방법은 `block_on` 함수를 이용하는 것입니다.

```rust,edition2018
# extern crate async_std;
use async_std::{fs::File, io, prelude::*, task};

async fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path).await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    Ok(contents)
}

fn main() {
    let reader_task = task::spawn(async {
        let result = read_file("data.csv").await;
        match result {
            Ok(s) => println!("{}", s),
            Err(e) => println!("Error reading file: {:?}", e)
        }
    });
    println!("Started task!");
    task::block_on(reader_task);
    println!("Stopped task!");
}
```

이 코드는 파일을 읽는 코드를 실행하기 위해 `async_std`에 내장된 런타임(역자 주: async_std::task를 이용한다는 뜻입니다)에 요청하는 방식입니다. 이제 안에서 밖으로 하나씩 살펴봅시다.

```rust,edition2018
# extern crate async_std;
# use async_std::{fs::File, io, prelude::*, task};
#
# async fn read_file(path: &str) -> io::Result<String> {
#     let mut file = File::open(path).await?;
#     let mut contents = String::new();
#     file.read_to_string(&mut contents).await?;
#     Ok(contents)
# }
#
async {
    let result = read_file("data.csv").await;
    match result {
        Ok(s) => println!("{}", s),
        Err(e) => println!("Error reading file: {:?}", e)
    }
};
```

가장 안쪽에 있는 `async` *블록*입니다. async 블록은 `async` 함수를 호출하는데 필요하며 컴파일러에게 모든 관련 명령어를 포함하도록 지시합니다. Rust에서 모든 블록은 값을 반환하는데 `async` 블록은 `Future` 타입의 값을 반환합니다.

좀 더 흥미로운 부분을 살펴봅시다.

```rust,edition2018
# extern crate async_std;
# use async_std::task;
task::spawn(async { });
```

`async` 바깥 블록을 보면 `spawn`이 나왔습니다. `spawn`은 `Future`를 받아서 `Task`에서 실행하게 합니다. 그러면 결과값으로 `JoinHandle` 타입이 나옵니다. Rust의 Future는 때때로 *cold Future*라고 불립니다. (자체로는 실행을 시작할 수 없어서) Future 실행을 지시하는 무언가가 필요하기 때문입니다. Future를 실행하려면 추가적인 정보 기재가 필요합니다. 작업이 실행 중인지 아니면 완료되었는지, 메모리에 배치되어 있는지 현재 상태가 어떤지 등이죠. 이런 추가적인 정보 기재 부분을 추상화한 것이 바로 `Task`입니다.

`Task`는 `쓰레드`와 비슷하지만 약간의 차이가 있습니다. 쓰레드는 운영체제 커널에서 조정하지만 `Task`는 프로그램에 의해서 조정됩니다. 대기해야 하는 상황이 생기면 프로그램 자체가 다시 `Task` 작동을 요청할 책임을 가지고 있습니다. 여기에 대해서는 다음에 조금 더 다룰 것입니다. 공통점도 많은데 예를 들어 `async_std`의 task도 쓰레드와 마찬가지로 이름과 ID를 가지고 있습니다.

여기서는 task를 `spawn`하면 task가 백그라운드에서 계속 실행된다는 것을 알면 충분합니다. `JoinHandle` 자체는 `Task`가 결과를 내놓을 때 완료되는 future입니다. `쓰레드`나 `join` 함수와 비슷하게 이제 프로그램(정확하게는 특정 호출 쓰레드)을 *블록*하고 task가 종료될때까지 기다리기 위해서 `block_on`을 호출할 수 있습니다.

## `async_std`의 Task

`async_std`의 Task는 가장 핵심적인 추상화된 개념 중 하나입니다. Rust의 `쓰레드`와 비슷하게 Task는 기초개념에 대한 실용적인 기능을 제공합니다. `Task`는 런타임과 관계가 있지만 그 자체로는 분리되어 있습니다. `async_std`의 Task는 여러 가지 좋은 속성들을 가지고 있기도 합니다.

- Task 전체를 하나의 단일 할당(single allocation)으로 처리합니다.
- 모든 Task는 결과값과 에러를 도출한 후 (Task에서 결과값으로 뱉는) `JoinHandle`을 통해 다른 Task에 전파(spawn)할 수 있는 *백채널*을 가지고 있습니다.
- 디버깅을 위한 유용한 메타데이터들을 가지고 있습니다.
- Task를 위한 로컬 스토리지도 지원합니다.

`async_std`의 Task API는 명시적으로 시작하는 런타임과는 무관하게 런타임의 설정과 해체 작업을 수행합니다.

## Blocking

`Task`는 잠재적으로 실행 쓰레드를 공유하면서 _병렬적으로_ 작동한다고 가정합니다. 즉, `std::thread::sleep`이나 Rust의 `std` 라이브러리의 io 함수같이 *운영체제 쓰레드*를 차단하는 작업은 그 쓰레드가 공유되는 모든 Task의 실행을 중단시킵니다. (데이터베이스 드라이버같은) 다른 라이브러리들도 비슷하게 작동합니다. *현재 쓰레드를 블로킹*하는 것은 그 자체가 나쁜 동작은 아니지만 `async-std`의 병렬적 실행모델과 잘 맞지 않습니다. 그러니까 이렇게는 쓰지 마세요.

```rust,edition2018
# extern crate async_std;
# use async_std::task;
fn main() {
    task::block_on(async {
        // this is std::fs, which blocks
        std::fs::read_to_string("test_file");
    })
}
```

이런 명령들을 섞어서 쓰고 싶다면 별도의 `쓰레드`로 블로킹 명령들을 보내는 걸 고려해보세요.

## 에러와 패닉

Task는 일반적인 패턴으로 오류를 보고합니다. 오류가 발생했을 때 출력 타입은 `Result<T, E>`이어야 합니다.

`패닉`의 경우 `패닉`을 처리하는 코드가 있는지 여부에 따라 대응이 달라집니다. 만약 처리 코드가 없다면 프로그램이 _중단됩니다_.

실제로는 `block_on`이 블로킹 컴포넌트에 패닉을 전달한다는 뜻입니다.

```rust,edition2018,should_panic
# extern crate async_std;
# use async_std::task;
fn main() {
    task::block_on(async {
        panic!("test");
    });
}
```

```text
thread 'async-task-driver' panicked at 'test', examples/panic.rs:8:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
```

생성된 Task가 패닉이 되어있는 동안 Task 는 중단됩니다.

```rust,edition2018,should_panic
# extern crate async_std;
# use async_std::task;
# use std::time::Duration;
task::spawn(async {
    panic!("test");
});

task::block_on(async {
    task::sleep(Duration::from_millis(10000)).await;
})
```

```text
thread 'async-task-driver' panicked at 'test', examples/panic.rs:8:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
Aborted (core dumped)
```

처음에는 이상하게 보일 수 있지만 생성된 Task에서 발생한 패닉을 조용하게 무시하는 것도 가능합니다. 지금의 동작은 생성된 Task에서 패닉을 포착했을 때의 대응방식을 사용자가 수정하여 변경할 수도 있습니다. 이를 통해 사용자는 패닉 처리 전략을 선택할 수 있습니다.

## 결론

`async_std`는 `std::thread`와 유시한 API로 작동하는 `Task` 타입을 제공하여 유용하게 사용할 수 있습니다. 이를 통해 구조화된 방식으로 에러와 패닉을 다룰 수 있습니다.

Task는 별도의 병렬처리가능한 단위이며 때로는 통신도 필요합니다. 그래서 `Stream`이 나오는 것이죠.

[tasks]: https://docs.rs/async-std/latest/async_std/task/index.html
