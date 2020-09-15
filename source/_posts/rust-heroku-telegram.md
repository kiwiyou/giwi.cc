---
title: Heroku와 Rust로 텔레그램 웹훅 봇 만들기
comments: true
date: 2020-09-15 16:52:21
tags:
    - rust
    - heroku
    - 텔레그램
category: 프로그래밍
---
다양한 기능을 가지고 있는 텔레그램에서 유독 개발자의 눈에 들어오는 것은 바로 _봇_ 기능이다.
자신만의 봇을 사용하려면 봇을 구동해줄 서버가 필요한데, 직접 서버를 마련하는 것보다는 다른 쪽에 맡기는 것이 아무래도 편하다.
Rust에 관심이 있는 텔레그램 사용 개발자(얼마나 될지..)를 위해 Rust로 텔레그램 웹훅 봇을 제작하고 Heroku에서 무료로 봇을 호스팅하는 법까지 써보려 한다.

# 봇 계정 생성하기

개발에 앞서 먼저 봇을 위한 계정을 생성해야 한다.
텔레그램 [@BotFather](https://t.me/BotFather)을 통해서 계정을 만들 수 있다.
이미 계정이 있다면 [다음 절](#프로젝트-생성)로~

![BotFather과 처음 대화한 경우](/img/rust-heroku-telegram-botfather.png)
BotFather과 대화하는 것이 처음이라면 위 사진처럼 start 버튼이 있을 것이다. 눌러서 대화를 시작하자.

![/newbot 명령어를 통해 봇을 생성하기](/img/rust-heroku-telegram-newbot.png)
`/newbot` 명령어를 입력해 봇을 만드는 작업을 시작한다.
처음으로 봇의 이름(대화창에서 보이는 이름)을 입력하고, 봇이 사용할 사용자명(@로 시작하는 이름으로 검색에 사용된다.)을 입력해주면 봇이 만들어진다.
그러면 BotFather가 장문의 메시지를 보내오는데, 둘째 문단에 보이는 `7034227848:AAFP...`가 봇 구동에 사용되는 고유 토큰이다.
여기서는 포스트를 쓰기 위해 공개했지만, 토큰을 통해 봇을 조종하므로 유출되지 않도록 해야 한다.
물론 유출이 의심될 시에는 BotFather을 통해 토큰을 재발급할 수 있다.

# 프로젝트 생성

원하는 디렉토리에서 아래 명령어를 통해 Rust 프로젝트를 생성한다.

```bash
$ cargo new rust-heroku-telegram
```
_`rust-heroku-telegram` 부분은 프로젝트 이름이므로 마음대로 바꿔도 좋다._

Cargo.toml에 텔레그램 봇을 만들기 위한 의존성을 추가한다.
텔레그램 봇을 위한 crate는 [여럿 있지만](https://lib.rs/keywords/telegram) 우리는 [tbot](https://lib.rs/crates/tbot)을 사용할 것이다.

```toml
[dependencies]
tbot = "0.6.5"
tokio = { version = "0.2.22", features = ["macros"] }
```

# 코드 작성

이어서 `src/main.rs`에 아래 코드를 입력하자.
메시지를 입력하면 똑같은 내용으로 답장을 해 주는 코드이다. ([tbot examples에서 일부 수정했다](https://gitlab.com/SnejUgal/tbot/-/blob/HEAD/examples/webhook.rs))
라이브러리 사용법을 더 자세히 알고 싶다면 [문서를 읽어보자.](https://docs.rs/tbot/0.6.5/tbot/)

```rust
use std::env;
use tbot::{prelude::*, Bot};

#[tokio::main]
async fn main() {
    let token = env::var("BOT_TOKEN").unwrap();
    let mut bot = Bot::new(token.clone()).event_loop();

    bot.text(|context| async move {
        let echo = &context.text.value;
        let call_result = context.send_message_in_reply(echo).call().await;

        if let Err(error) = call_result {
            eprintln!("Telegram Error: {}", error);
        }
    });

    let url = env::var("WEBHOOK_URL").unwrap();
    let port = env::var("PORT").unwrap().parse().unwrap();
    bot.webhook(&url, port)
        .accept_updates_on(format!("/{}", token))
        .ip("0.0.0.0".parse().unwrap())
        .http()
        .start()
        .await
        .unwrap();
}
```

위 코드에서 BOT_TOKEN 환경변수로는 봇 계정을 생성할 때 받은 길다란 토큰 값을 넣어주어야 한다.
WEBHOOK_URL은 텔레그램의 웹훅을 받게 될 주소로, 우리는 보안상의 이유로 heroku가 제공해 주는 url에 토큰을 붙여서 사용할 것이다.
PORT는 웹훅 서버를 바인딩할 포트인데, heroku 환경에서 자동으로 설정되는 환경변수다.

다음으로 heroku가 우리 프로그램을 웹서버로 인식할 수 있도록 Procfile을 만들어 줘야 한다.
`Procfile` 파일에 아래 내용을 입력해주자.

```
web: target/release/rust-heroku-telegram
```
만약 위에서 프로젝트 이름을 다르게 지었다면, `rust-heroku-telegram` 부분을 프로젝트 이름으로 바꿔주어야 한다.

# heroku를 통해 deploy

heroku-cli가 없다면 [여기서 다운로드할 수 있다.](https://devcenter.heroku.com/articles/heroku-cli)

먼저 heroku create를 통해 heroku 앱을 새로 만든다.

```bash
$ heroku create rust-heroku-telegram
Creating ⬢ rust-heroku-telegram... done
https://rust-heroku-telegram.herokuapp.com/ | https://git.heroku.com/rust-heroku-telegram.git
```

다음으로 Rust 프로젝트를 빌드하기 위해 rust heroku buildpack을 설치한다.

```bash
$ heroku buildpacks:set emk/rust
Buildpack set. Next release on rust-heroku-telegram will use emk/rust.
Run git push heroku main to create a new release using this buildpack.
```

이제 환경변수를 설정할 차례다.
WEBHOOK_URL의 경우 `(heroku 앱의 url)/(토큰)` 형식으로 설정해준다.

```bash
$ heroku config:set BOT_TOKEN=703427848:AAFPVZ0N4gxD2ECu4SM6qtxtmnXWeC13zFc
Setting BOT_TOKEN and restarting ⬢ rust-heroku-telegram... done, v3
BOT_TOKEN: 703427848:AAFPVZ0N4gxD2ECu4SM6qtxtmnXWeC13zFc

$ heroku config:set WEBHOOK_URL=https://rust-heroku-telegram.herokuapp.com/703427848:AAFPVZ0N4gxD2ECu4SM6qtxtmnXWeC13zFc
Setting WEBHOOK_URL and restarting ⬢ rust-heroku-telegram... done, v4
WEBHOOK_URL: https://rust-heroku-telegram.herokuapp.com/703427848:AAFPVZ0N4gxD2ECu4SM6qtxtmnXWeC13zFc

$ heroku config
=== rust-heroku-telegram Config Vars
BOT_TOKEN:   703427848:AAFPVZ0N4gxD2ECu4SM6qtxtmnXWeC13zFc
WEBHOOK_URL: https://rust-heroku-telegram.herokuapp.com/703427848:AAFPVZ0N4gxD2ECu4SM6qtxtmnXWeC13zFc
```

이제 git을 통해 heroku remote에 변경 사항을 저장하면 자동으로 빌드가 시작되고, deploy가 완료된다.

```bash
$ git add .
$ git commit -m 'My first bot 🥳'
$ git push heroku master
# ...
-----> Launching...
       Released v5
       https://rust-heroku-telegram.herokuapp.com/ deployed to Heroku
```

![성공적으로 봇이 작동한다.](/img/rust-heroku-telegram-done.png)
