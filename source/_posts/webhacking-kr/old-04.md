---
title: Webhacking.kr old-04
date: 2020-09-08 17:13:28
tags:
    - webhacking.kr
    - php
    - bruteforce
category: 워게임
---
# 관찰

![old-04-reconnaissance](/img/old-04-reconnaissance.png)

해시 값처럼 보이는 문구가 있고, 비밀번호 입력부와 제출 버튼이 있다.
위 문구를 해석해서 아래에 쓰는 것으로 보이는데, 사이트를 새로 고칠 때마다 해시 값이 바뀌는 것을 볼 수 있다.

# 코드 분석

```php
<?php
  sleep(1); // anti brute force
  if((isset($_SESSION['chall4'])) && ($_POST['key'] == $_SESSION['chall4'])) solve(4);
  $hash = rand(10000000,99999999)."salt_for_you";
  $_SESSION['chall4'] = $hash;
  for($i=0;$i<500;$i++) $hash = sha1($hash);
?>
<!-- 중략 -->
<tr><td colspan=3 style=background:silver;color:green;><b><?=$hash?></b></td></tr>
```

10000000부터 99999999 중 숫자를 무작위로 뽑아 원본 문자열을 만들고, [SHA-1](https://en.wikipedia.org/wiki/SHA-1)을 500번 적용하여 보여준다.
아무리 생각해도 이상했지만 원본 문자열을 찾으려면 사전을 만들어서 검색하는 수밖에 없었다.

```rust
use indicatif::{ParallelProgressIterator, ProgressBar, ProgressStyle};
use rayon::prelude::*; // 병렬 처리
use sha1::{Digest, Sha1};
use std::io::{BufWriter, Write};

fn main() {
    let style = ProgressStyle::default_bar()
        .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos:>9}/{len:9} {msg}")
        .progress_chars("##-"); // 진행도를 보여줘야 속이 안 터지는 타입.
    let bar = ProgressBar::new(40000000).with_style(style.clone());
    bar.set_message("gen...");
    let hashes: Vec<_> = (10000000u32..=39999999)
        .into_par_iter()
        .progress_with(bar)
        .map(|i| {
            let mut s = i.to_string();
            s.push_str("salt_for_you");
            (i, s)
        })
        .map(|(i, s)| {
            let mut hash = s;
            for _ in 0..500 {
                hash = format!("{:x}", Sha1::digest(hash.as_bytes()));
            }
            (i, hash)
        })
        .collect();
    let file = std::fs::File::create("table.txt").unwrap();
    let mut writer = BufWriter::new(file);
    for (i, hash) in hashes.iter() {
        writeln!(writer, "{},{}", i, hash).unwrap();
    }
}
```
앞쪽 4천만개만 뽑아서 사전을 만들었는데 이 정도면 금방 답을 찾을 수 있다.
사전 생성에는 12분 정도 걸렸다.

## SHA-1의 취약성

SHA-1은 해시 충돌을 충분히 빠르게 구할 수 있다는 것이 밝혀졌지만 그래봤자 나의 귀여운 노트북으로는 어림도 없는 계산량이므로 차치하고...
[SHA-1 알고리즘의 의사 코드](https://en.wikipedia.org/wiki/SHA-1#SHA-1_pseudocode)를 보면 해시에 그리 오랜 시간이 걸리지 않으므로 사전 공격에 약한 편이다.
(물론 salt를 사용한다면 또 달라지겠지요)
따라서 비밀번호를 저장할 때 SHA-1보다는 비밀번호를 저장하기 위해 만들어진 [argon2](https://argon2.online/), bcrypt 등을 사용하는 것이 좋다.

## 여담

위의 사전 생성 코드는 숫자와 해시 문자열을 저장한다.
원본 문자열에는 salt_for_you가 붙는데 답을 입력할 때 빼먹어서 삽질을 조금 했다...
이것만 조심하면 1천만개(약 3분)정도만 뽑아내도 몇 번 시도해서 금방 통과할 수 있을 듯하다.
