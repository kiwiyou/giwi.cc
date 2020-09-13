---
title: 남의 코드 리팩터링하기
comments: true
date: 2020-09-14 00:07:58
tags:
category: 프로그래밍
---
Rust 관련 채팅방에서, 아는 분께서 Rust에 입문하신다길래 코드를 리팩터링해드리기로 했다.
명목상으로는 더 나은 코드가 무엇인지 생각해볼 수 있다는 이유였지만, 사실 다른 사람의 코드 구조를 바꿀 수 있다는 흔치 않은 기회이기에 이기적으로 도전해보기로 했다.
(PR 내용을 보면 아시겠지만 저는 Author로 본인을 추가하는 악질 개발자입니다.)

건드리게 된 레포지토리는 [텔레그램 다나와 가격 변동 알리미 봇](https://github.com/B4TT3RY/danawa_price)이다.

# 원래 코드 훑어보기

개발자는 코드로 대화한다고 했던가.
텔레그램 API를 포함해서 프로젝트에 쓰인 모든 crate는 이미 몇 번 써 본 적이 있기 때문에 따로 봇을 사용해보진 않고 코드를 보러 갔다.

## main.rs

먼저 맨 위에 보이는 `extern crate`들이 눈에 띄었다.
![불필요한 extern crate가 쓰인 모습](/img/refactor-others-code-extern-crates.png)
Rust 2018에서는 `extern crate`가 전혀 필요 없고, 모두 `use`만 써서 crate를 사용할 수 있다.
[정확한 설명은 The Rust Reference에서 볼 수 있다.](https://doc.rust-lang.org/reference/items/extern-crates.html#extern-prelude)
이는 큰 작업이 아니기 때문에 나중에 [코드 포매팅과 함께 진행하기로 했다.](https://github.com/B4TT3RY/danawa_price/pull/1/commits/dea5c6ef3eced865b6589d3116ec07ad4081dea3)

그다음으로는 html 파싱에 사용하는 struct가 보였다.
![html 파싱에 사용하는 struct의 정의부](/img/refactor-others-code-html-struct.png)
여기선 두 가지 문제점을 찾을 수 있었다.

- 필드의 정의가 필드의 역할과 맞지 않는다.
  - `card_price`, `cash_price`는 _분명 숫자여야 하는_ 가격 정보를 담는데 String 자료형이다.
  - `#[html(default = "")]`를 통해 정보가 없는 경우를 "문자열로" 처리하고 있다.
필드의 정의가 역할과 맞지 않는다는 것은 프로그래머가 코드를 보고 무슨 역할인지 한 눈에 알 수 없다는 뜻이다.
자고 일어나보니 무슨 코드인지 모르는 사태가 발생할 수 있다는 것이다.

- 한 모듈이 가지는 역할이 너무 많다.
  - main 모듈은 전체 흐름을 담당하고 있다.
  - main 모듈은 주요 기능인 다나와 검색에 대한 세부 구현 _또한_ 포함하고 있다.
한 모듈의 역할이 많은 문제는 각 역할 사이의 의존성을 높이게 된다.
예를 들어 다나와 사이트에서 가격 표기 방식을 "3,000원" 방식에서 "3/000원" 같은 방식으로 바꾼다면 검색 부분이 아니라 웬 생뚱맞은 price_distance 부분에서 고쳐야 한다.
이는 특히 아래에 쓸 가격 저장 부분에서 두드러지는 문제다.

이 문제는 `danawa` 모듈을 분리하여 처리하기로 했다.
여기에는 아래에 있는 `async fn danawa`도 포함된다.

다음으로 `Data.toml` 파일을 읽어들여 `HashMap`으로 변환하는 코드가 있어서 다시 파일로 내보내는 부분도 같이 봤다.
![Data.toml 입력과 역직렬화](/img/refactor-others-code-data-read.png)
![Data.toml 출력과 직렬화](/img/refactor-others-code-data-write.png)

방금 봤던 "너무 많은 역할" 문제점을 여기서 찾을 수 있다.

main의 역할에 정보 입출력의 세부 구현까지 포함되는 것이다.
이번엔 `price` 모듈로 분리했다.

`async fn danawa`의 경우 settings를 계속 넘겨줘야 하는데, 다른 정보가 노출됨은 물론이고 불러주는 쪽에서 settings가 없을 수도 있다.
그래서 필요한 정보만 미리 저장해 두는 struct를 하나 만들기로 했다. 후에 설명할 텔레그램 API 부분도 이렇게 처리했다.

`fn price_distance`도 입력 파싱, 차이 계산, 출력 포매팅 등 구성이 복잡하다.
enum을 통해 가격 변동폭을 추상화하고 포매팅은 따로 구현해서 main에서는 `format!`을 사용하도록 했다.

settings.rs는 따로 볼 부분이 없어서 바로 telegram.rs로 넘어가자.

## telegram.rs

보통 코드를 위에서 아래로 보긴 하지만 `fn syntax` 코드가 너무 강렬해서 시선을 빼앗겼다.
![fn syntax에 잘못 쓰인 replace](/img/refactor-others-code-fn-syntax.png)

여기서 쓰인 replace는 계속해서 String을 복사한다. (reference인 `&str`를 받아서 owned type인 `String`를 주니 복사가 일어날 수밖에 없다.)
replace하는 문자의 가짓수만큼 전체 문자열의 복사가 일어나니 속도 저하가 있다. (현재 사용처 한정으로 문자열이 길지 않아 큰 차이는 없다.)
그래서 문자를 하나하나 살피며 이스케이프해주는 방식으로 처리하기로 했다.

그나저나 이 부분을 처음 코딩할 때 정신을 놓고 했는지 제대로 돌아가지 않는 코드를 기여했다...
PR을 넣을 때는 테스트를 무조건 거쳐야 하는데 지인 코드랍시고 제대로 테스트를 돌리지 않은 점을 반성한다.

위쪽은 텔레그램 API를 사용하는 부분인데, `api_link`와 `chat_id` 부분이 중복되는 것을 확인할 수 있다.
![인자로 넘어가는 api_link와 chat_id가 중복되고 있다.](/img/refactor-others-code-telegram-api.png)

이 두 요소를 미리 저장해뒀다가 API를 사용할 때 꺼내 쓸 수 있게끔 struct를 구성하기로 했다.

# 수정 결과

2~3시간 정도 걸려서 전체 코드를 고친 것 같다.
각 파일별로 어떻게 수정이 되었는지 살펴보자.

## telegram.rs

```rust
pub struct Sender {
    api_link: String,
    chat_id: String,
}

impl Sender {
    pub fn new(bot_token: &str, chat_id: &str) -> Self {
        Self {
            api_link: format!("https://api.telegram.org/bot{}/", bot_token),
            chat_id: chat_id.to_string(),
        }
    }

    pub async fn send_message(&self, message: &str) -> reqwest::Result<()> {
        // 생략
    }

    pub async fn set_chat_description(&self, description: &str) -> reqwest::Result<()> {
        // 생략
    }
}
```
`api_link`와 `chat_id`를 `Sender` struct로 떼어냄으로써 값의 재사용을 손쉽게 하였고, `send_message`와 `set_chat_description`을 메서드로 넣었다.
`new`를 통해서 `Sender`를 생성한 이후 내부 저장 방식이 공개되지 않기 때문에 내부 로직 변화에도 쉽게 대처할 수 있다.

## price.rs

```rust
pub struct PriceData {
    pub card_price: Option<i32>,
    pub cash_price: Option<i32>,
}

pub struct PriceStorage {
    price_map: HashMap<String, PriceData>,
}
```
먼저 가격 정보가 카드 가격과 현금 가격으로 항상 나뉘므로 `PriceData`로 묶어 관리했다.
또한 `PriceStorage`를 통해서 저장 방식 등을 숨겨 기능 분리를 달성했다.

따로 적지는 않았지만 기존에 `unwrap`만 사용했던 오류 처리를 `Result`를 통한 전파로 변경했다.

## danawa.rs

```rust
pub struct ProductInfo {
    pub product_name: String,
    pub url: String,
    pub price: PriceData,
}

pub struct Searcher {
    url: String,
}

impl Searcher {
    // ...

    pub async fn get_product_info(&self, product_code: &str) -> Result<ProductInfo, SearchError> {
        // 생략
    }
}
```
위쪽 [telegram.rs](#telegram-rs)와 같은 방법으로 `url`을 보관하는 `Searcher`을 만들고, 검색에 사용되는 `DanawaData`와는 다른 `ProductInfo`를 만들었다.
`DanawaData`는 웹페이지의 구조를 나타내는 struct, `ProductInfo`는 가격 정보를 나타내는 struct, 이렇게 의미를 부여하면 역할이 꼬이지 않을 거라 생각했다.

## main.rs

main은 거의 새로 짰다 할 정도로 많은 수정이 있었는데, 대부분은 price 모듈을 작성하면서 어쩔 수 없이 수정했던 것이다.

그 외에는 기존에 `price_distance`로 가격 정보를 나타내는 문자열을 받아 바로 포매팅했던 것과 달리, 가격 변동을 나타내는 `Difference`를 만들어 정보를 보관한 것이 있다.
```rust
enum Difference {
  Error,
    Up(i32),
    Stay,
    Down(i32),
}

impl std::fmt::Display for Difference {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Difference::Error => f.write_str("정보없음"),
            Difference::Stay => f.write_str("-"),
            Difference::Up(up) => write!(f, "▲{}원", up.to_formatted_string(&Locale::ko)),
            Difference::Down(down) => write!(f, "▼{}원", down.to_formatted_string(&Locale::ko)),
        }
    }
}

fn price_difference(prev: Option<i32>, now: Option<i32>) -> Difference {
    match (prev, now) {
        (_, None) => Difference::Stay,
        (None, _) => Difference::Error,
        (Some(prev), Some(now)) => {
            let diff = now - prev;
            if diff > 0 {
                Difference::Up(diff)
            } else if diff < 0 {
                Difference::Down(-diff)
            } else {
                Difference::Stay
            }
        }
    }
}
```
price.rs에서 가격의 표현을 변경한 덕분에(`String`에서 `Option<i32>`로) `price_difference`의 코드가 간결해졌고, enum variant 이름을 통해서 어떤 변동이 일어났는지 바로 확인할 수 있는 등 가독성이 높아졌다.
또한 `std::fmt::Display`를 구현하여 가격 변동을 표현하는 로직을 분리하였다.
다시 말하지만 이렇게 역할을 나눔으로써 수정 시에 수정해야 할 위치를 찾는 데 어려움을 겪거나 위치를 잊어버리는 등의 실수를 피할 수 있다.

# 후기

__재밌었다!__
다른 사람의 코드를 보니까 자신의 코드를 보는 것과는 또 다르게 시험문제를 푸는 듯한 느낌이 들었다.
덕분에 계속 고민할 기회가 되었고, 더 나은 구조를 고민하게 된 것 같다.
한편 PR을 넣은 이후 오작동 사례가 많이 보여서 테스트를 안 한 것을 후회하고 또 후회하는 중이다.
마치 "이렇게 잘 해줬으니까 알아서 버그 잡아"처럼 무책임한 기여자가 되지 않도록 조심해야지.
