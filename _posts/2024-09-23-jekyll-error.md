---
layout: post
title:  "Jekyll 빌드가 안되서..."
date:   2024-09-23 00:10:00 +0900
author: Changbae Bang
tags: [버전업, 에러확인, jekyll, build, ]
---

# 들어가면서
버전 올려서 고쳤다.


# 발단
jekyll Build 가 실패 했다.

```
Resolving dependencies...
ffi-1.17.0-x86_64-linux-musl requires rubygems version >= 3.3.22, which is
incompatible with the current version, 3.0.6
Error: Process completed with exit code 5.
```

뷰티뷸 지킬 업데이트 안했더니 망했구나 해서 그냥 냅다 돌려 봤다.
마침 또 [6.0.1](https://github.com/daattali/beautiful-jekyll/blob/6.0.1/.github/workflows/ci.yml)이 있었다.

# 전개
안된다. 그러면 그렇지

```
sass-embedded-1.58.3-x86_64-linux-musl requires rubygems version >= 3.3.22,
which is incompatible with the current version, 3.0.6
Error: Process completed with exit code 5.
```

흠 어쩌나, 이때부터 뷰티뷸 지킬의 영역이 아니다.

지킬 도커 버전 확인한다.
지킬 쪽 상황 확인한다.

일단 [alpine 리눅스에서 jekyll 실행 시 sass-embedded-1.57.1 에러](https://mumbi.net/jekyll/fix-jekyll-build-error-in-alpine-linux/)이 있다고 한다.

하.. 난 도커파일도 만들기 싫은데 어쩌나..

# 절정

아!!! 막 그냥 버전 때려 써본다.
3.8 부터 천천히 올려보지 뭐....
아시다 싶이 그렇게 문제가 해결되지 않는다.

# 결말
4가 이러쿵 저러쿵 버전 올리고 맞추고 저쩌구 해서 해결이 되었음을 확인하며,  
무의미한 숫자 올리기를 멈춘다.
[Segfault when running Jekyll 4 in Alpine](https://github.com/jekyll/jekyll/issues/7801#issuecomment-525609325)  
흠 찝찝할 수 있지만 도전!

```
Bundle complete! 3 Gemfile dependencies, 34 gems now installed.
```


# 마치며
지킬에서 나올때가 되나 싶다.  
구글 에드센스 떼어내고 휙 이사를 가야 하나 싶다.  
확 마 넥스트로 처리해볼까 싶기도 하고  
리엑트로 그냥 구현할까 싶기도 하지만  
나는 단지 겸허히 오늘 아침 출근해야 하는 직장인일 뿐  
더도 덜도 아니다.   
