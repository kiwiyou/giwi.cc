---
title: Webhacking.kr old-01
date: 2020-09-08 04:25:00
tags:
    - webhacking.kr
    - php
    - type-juggling
category: 워게임
---
# 관찰

![old-01-reconnaissance](/img/old-01-reconnaissance.png)

페이지 소스를 둘러보았을 때 사용자와 상호작용하는 요소는 없어 보인다.
view-source를 통해 소스 코드를 살펴봐야겠다.

# 코드 분석

```php
<?php
  if(!is_numeric($_COOKIE['user_lv'])) $_COOKIE['user_lv']=1;
  if($_COOKIE['user_lv']>=6) $_COOKIE['user_lv']=1;
  if($_COOKIE['user_lv']>5) solve(1);
  echo "<br>level : {$_COOKIE['user_lv']}";
?>
```

`user_lv` 쿠키 값을 5보다 크면서 6보다 작게 만들면 `solve(1)`이 실행된다.
평소에 실수(実數)를 쓸 일이 거의 없었던 터라 순간 머리가 굳었지만 금방 풀 수 있었다.

## 여담

주변에서 php의 악명이 높다는 이야기를 많이 들어서(js의 == 비교처럼) 숫자가 아닌 다른 길을 찾아보려고 했다.
하지만 [비교 결과표](https://www.php.net/manual/en/language.operators.comparison.php)에서 마땅한 값을 찾지 못했다.
