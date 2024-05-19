---
layout: post
title:  "선별적(?) package 설치가 될까?"
date:   2024-05-19 19:00:00 +0900
author: Changbae Bang
tags: [FE, 개발, npm,  resolutions, ]
---

# 들어가면서
나는 큰 슬럼프를 겪고 있다. 나는 과연 기술력이 있는가? 무언가 만들어낼 능력이 있는가? 에 대한 물음에 제대로 답하지 못하게 되었다.  

최근 회사에서 사용하는 중요 package 가 이름과 배포 방식을 달리 하게 되었다. 이름이 바뀐 것이야 구분도 간단하고 사용도 변경 요소도 바뀐 것은 없어 기존 코드를 거의 손댈 필요는 없다.  
하지만 typescript 설정의 compile target 이 `esnext` 로 변경하여 개발 되었다.  

> 참고  
[tsconfig - target](https://www.typescriptlang.org/tsconfig/#target)  
Default 는 ES3

원래 사용하던 모듈들은 'es5` 로 구성되어 있었다. 뭐 당연히 설치가 부드럽게 되지 않았고, 배포 일정이 코앞이라 에전 package 에서 처리해서 예전 package 의 새 버전을 설치하게 하는 식으로 처리를 하였다.  

예상 되었던 일이긴 했는데, 일단 그 공통 모듈의 릴리즈가 이렇게 빨리 올줄 몰랐고(그 전에 꾸닥 처리를 하려고 했는데...) 사실 막을 수도 없었다.(이미 PR 들은 main 을 이미 장악..)  

사실 이럴 땐 기존 package 를 fork 해서 처리하다가 적응이 되었을 때 합하는 것도 좋은 방법인데, 그렇게 처리할 수가 없었다. 정확히 말하면 내가 조절할 수가 없었다.  

이로 인해서 극심한 고민에 빠지게 되었다.  

나는 뭐하는 사람일까?  

# 그냥 다른 고민하자
어떤 코드가 오픈 소스인데 또 어떤 부분은 비공개 요소라 배포하지 못하는 상황에 대해서 생각해보게 되었다.  
이제는 생각 나지도 않는 webOS 시절을 떠올려 본다.  
어쨌든 목적별로 라이브러리 목록도 다르고 라이브러리의 위치도 다르다. (TV 인지 Open Source 버전인지..)  

사실 이런일 하기도 귀찮고, 빌드도 CMake 도 다 힘들어서 Web 으로 넘어온 것이긴 한데...   
사실 더하면 더 했지 덜하지 않는다. 단지 컴파일만 안할 뿐이다.  
심지어 타입스크립트로 넘어오면서 그 컴파일이라는 것도 한다. 물론 예전 만큼의 고통은 아니지만  

어쨌든 제품에 A 는 B 라는 컴포넌트가 상황에 따라 설치 되고 안되고 하는 상황을 고민해 봤다.  

물론 A, B 를 단일 저장소로 처리할 수는 없을 것이다.  
A 는 공개고 B 는 비공개이어야 한다는 원칙을 지키려면 우선 저장소가 분리 되어야 한다.  

# 그럼 설치 할 때 깔거나 말거나 하면 되지 않는가?
우선 yarn 으로 설치할 때 `dependencies` 에 없으면 설치가 멈춘다. 그리고 yarn 으로 build 할 때 목록에 없는 컴포넌트를 import 하면 build 가 되지 않는다.  

하나씩 풀어보자.  

## optionalDependencies

[optionalDependencies](https://yarnpkg.com/configuration/manifest#optionalDependencies) 은 그런거 아니다. 찾아 보지 말자.  
> Set of dependencies that Yarn should only try to install if the os/cpu/libc fields match those of the host platform.

## dependencies 그리고 resolutions
[resolutions](https://yarnpkg.com/configuration/manifest#resolutions) 이것으로 하면 가능성이 있다.  
> Override the resolutions of specific dependencies.  

## 그러면?
[package.json](https://github.com/changbaebang/next-package-test/commit/d23688722738f0225153e5a4239bc6eca816878a#diff-7ae45ad102eab3b6d7e7896acd08c427a9b25b346470d7bc6507b6481575d519) 에서 상세한 내용을 볼 수 있다.  

`react-my-icon` 이라는 비공개 저장소가 있으나, 실패하면 다들 잘 아는 `react-icon` 으로 넘어가도록 설정하였다.

``` package.json
  "dependencies": {
    "react-my-icons": "^4.3.1"
  },
  "resolutions": {
    "react-my-icons": "npm:react-icons@^4.3.1"
  },

```

원래는 아래 코드와 같이 import 를 해보면서 처리를 하려고 하였으나, 빌드 자체가 되지 않아서 이렇게 dynamic 과 import 를 가지고 처리하지 않아도 되게 되었다.  

```
const Empty = () => <></>;

const CustomFaBars = dynamic(async () => {
  try {
    const icon = await import('react-my-icons/fa');
    return icon.FaBars;
  } catch (error) {
    return Empty;
  }
});
```

대신 단지 그저 있냐 없냐만 처리하기에 빌드 에러 등등의 세부 사항은 어떻게 하기가 어렵다.  

원격에 있는 모듈을 마지 옆에 있는 듯 선별적으로 스르륵 가져오는 [Module Federation](https://webpack.kr/concepts/module-federation/)을 생각하기 전에 더 단순한 방법으로 처리하는 방안을 생각해보았다.  

Module Federation 역시 그 package 의 상태와 상황에 맞게 어쩔 수는 없으니... 가성비상은 package 이름에 별명을 붙이고 `resolutions` 으로 처리하는 것이 더 가성비가 있을 수 있다고 생각해 볼 수도 있을 것 같다.  

# 그럼 관리는 어떻게 해야 할까?
이 두 package 를 단일 저장소에서???  
안된다. 하나는 fallback 곧 공개 될 수 있는 것이고 하나는 아니다.  
C++ 등 많이 할 때 처럼 header 만 공개하고 내부는 감추는 등을 해야 하지 않을까 싶다.  

`react-my-icons` 과 `react-icons` 에 외부에서 사용하는 type 혹은 interface 는 다 공개를 해야 할 것이다.  
이러한 저장소를 공개로 하나 만들고, `react-icons` 에서 이 저장소를 기준으로 열심히 개발하고, `react-my-icons` 에서도 동일하게 개발을 하면 될 것 같다.  
다만 한눈에 안보이니 더 열심히 뛰어야 겠지.  

# 이 전략은 현재 적용 가능한가?
아쉽지만 현재 재직하고 있는 회사에서는 그러기가 어렵다.  
라이브러리에서 얼마만큼 어떤 방식으로 `export` 하는 부분부터 정리를 해봐야 할 것 같다. 그리고 나서 어떤 부분은 비공개, 공개를 해야 할지 선택을 해야겠지만, 현재로서는 다 공개이다.  
package 의 존재 여부가 아니라 build 나 install 시점에서 오류가 나면 똘똘하게 `package.json` 자체를 바꾸어서 빌드하는 CD/CI 수준에서의 처리가 필요할 것 같다.  

# 마치면서  
요 몇달 정말 진지하게 Technical Writer 라든가 Developer Relations 등의 전직을 고민했었다.  
그 이유로 나는 현재 살아있는 기술을 가진 기술자가 아니라고 판단했었다.  
하지만 그러한 직군 역시 나를 받아주느냐? 그렇지도 않다.  
그렇다면 다시 한번 기술력을 잘 끌어 올려 기술자로 일을 할 수 있는 상황으로 복귀를 해야 겠다는 생각을 다시 해본다.  

`폼은 일시적이지만 클래스는 영원하다` 리지만 나의 클래스가 어떨지는 요새 크게 의심이 된다. 그럼에도 아이들이 크는 등 나의 가족을 이어나기기 위한 비용은 필요하다. 그리고 내가 할 수 있는 일이 없다. 다른 돈 벌 수 있는 일이 없다. 배운 기술이 이게 다다.  

`우린 예전에 끝났어. 돈때문에 하는 거지` 라는 오아시시의 명언이 나에게 더 현실 적이지만, 노엘 겔러거가 지난 해 JTBC 와의 인터뷰를 할 때는 [돈보다 중요한 건 노래](https://news.nate.com/view/20231203n14555) 라고 했었다.  
나 역시 진중하게 `돈도 좋지만, 개발을 하고 그 개발한 것이 어떻게 저렇게 다음 세대의 삶에 좋은 영향을 끼치게 하자` 라를 생각을 잊지 않으려 한다.  

