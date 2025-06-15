next.js 는 어떻게 코드 스플리팅과 트리쉐이킹을 지원해주는걸까?

next.js는 리액트 프로젝트와는 달리 webpack.config.js 파일 대신 next.config.js 파일에서 웹팩 설정을 다를 수 있다.

next.js에서 기본적으로 제공해주는 웹팩 설정 위에서, 커스텀을 하는 형식이다.

웹팩 설정을 만져볼 때, 최적화 할 수 있는 부분이 코드 스플리팅과 트리쉐이킹이다.

하지만 next.js에서 이러한 최적화를 해보려고 하면, 내부적으로 이미 잘 지원이 되고 있어서 개발자가 추가로 건들 부분이 없다는 내용이 많다.

내부적으로 어떻게 되어있길래 개발자가 안해도 되는지 같이 알아보자.

### 코드 스플리팅

코드 스플리팅은 애플리케이션의 모든 코드를 하나의 큰 번들로 묶는 대신, 필요한 시점에 필요한 코드만 로드할 수 있도록 여러 개의 청크(Chunk)로 분리하는 전략이다.

Next.js는 페이지 기반 라우팅을 사용하기 때문에 각 페이지별로 코드를 자동으로 분할한다. 이 외에도 Webpack의 `splitChunks` 설정을 통해 공통 모듈도 효율적으로 분리하고 있다.

[https://github.com/FESIStudy/W19_BUNDLING_TOOL/tree/main/채유빈](https://github.com/FESIStudy/W19_BUNDLING_TOOL/tree/main/%EC%B1%84%EC%9C%A0%EB%B9%88)

위 글을 참고하여 Next.js 내장 웹팩을 확인해보았다. 그중, webpack-prod-client.json의 내용중에 이런 내용이 있다.

```jsx
"splitChunks": {
  "cacheGroups": {
    "framework": {
      "chunks": "all",
      "name": "framework",
      "priority": 40,
      "enforce": true
    },
    "lib": {
      "priority": 30,
      "minChunks": 1,
      "reuseExistingChunk": true
    }
  },
  "maxInitialRequests": 25,
  "minSize": 20000
}
```

해당 내용에 대해 하나씩 알아보자면, 

splitChunks은 코드 스플리팅의 핵심 옵션이다. Webpack은 기본적으로 모든 JS를 하나의 bundle.js로 만들 수 있지만, 이렇게 한다면 매번 전체 코드를 다운로드 해야하기에 비효율적이게 된다. 그래서 splitChunks 옵션을 통해 공통 모듈을 별도 파일로 분리해서 브라우저가 캐시할 수 있게 해줄 수 있다.

<aside>
💡

여기서 잠깐! 개발 모드에서는?

개발 모드에서는 빠른 빌드 속도와 간단한 디버깅, 그리고 HMR(Hot Module Replacement)를 위해 코드 스플리팅을 의도적으로 끄게 된다. 즉, 코드 스플리팅이 잘 이루어지는지 확인하기 위해서는 빌드 모드를 실행해야 한다.

</aside>

- framework
    - **React, ReactDOM, Next.js core 등 핵심 프레임워크**가 여기에 들어감
    - chunks
        - all(범용적): 동기 + 비동기(import()) 모듈까지 **모두 청크 분할 대상으로 지정 → 메인 코드든 dynamic import든 모두 분할 후보가 됨**
        - initial: 초기 로딩만
        - async: dynamic import만
    - name: "framework" → 분리된 파일 이름이 framework-[hash].js가 됨
    - enforce: true → 다른 조건과 무관하게 이 룰을 강제함
    - priority: 40 → 여러 룰이 겹칠 경우 우선순위가 가장 높음
    - React는 거의 바뀌지 않기 때문에 캐시에 두고 계속 재사용 가능 → **네트워크 부하 감소, 초기 렌더 속도 향상**
- lib
    - **node_modules에 있는 기타 라이브러리 (예: axios, lodash 등)가 여기에 들어감**
    - minChunks: 1 → 한 번 이상 사용되면 분리 가능성 있음
    - reuseExistingChunk: true → 동일한 모듈이 이미 청크에 있다면 그걸 재활용함
    - 공통 라이브러리를 한 번 로딩하면 여러 페이지에서 **중복 없이 재사용 가능**
- minSize: 20000 → **20KB 이상만 청크 분할** (너무 작은 건 합쳐서 요청 수 최소화)
- maxInitialRequests: 25 → 초기 로딩 요청 수 제한으로 네트워크 과부하 방지

- runtimeChunk
    - Webpack 런타임 코드가 `webpack.js`라는 별도 청크로 분리됨
    - 이 덕분에 캐시 효율 향상

이렇게 설정을 잘 해놓으면, 빌드 결과물은 아래처럼 나오게 된다.

```jsx
.next/
└── static/
    └── chunks/
        ├── framework-[hash].js    ← React, ReactDOM 등
        ├── lib-[hash].js          ← axios, lodash 등 node_modules 기반 공통 라이브러리
        ├── webpack-[hash].js      ← Webpack 런타임 코드
        ├── main-[hash].js         ← 공통 초기화 로직 (Next.js, client bootstrap 등)
        ├── pages/
        │   ├── index-[hash].js    ← 각 페이지별 JS (자동 코드 스플리팅)
        │   └── about-[hash].js
        └── [dynamic]-[hash].js    ← `import()`로 동적 로딩되는 컴포넌트
```

이렇게 잘 쪼개진 청크들은 어떻게 활용이 될까? 브라우저에서 캐시 활용 방법에 대해 알아보자.

1. 브라우저 캐시 활용 흐름

Next.js는 해쉬 번호를 통해 정적 리소스 캐시를 똑똑하게 활용한다.

브라우저에 첫 방문 시, 아래처럼 모든 청크를 다운로드 한다. → 해시 번호 바꾸기

```vbnet
GET /framework-ab12cd34.js   → 200 OK
GET /lib-ef56gh78.js         → 200 OK
GET /main-mn34op56.js        → 200 OK
GET /index-qr78st90.js       → 200 OK
```

하지만, 이후 페이지 코드, 즉 컴포넌트 등의 수정이 이루어졌을 때는 해쉬 번호를 보고 수정이 이루어졌는지 판별하여 필요한 파일만 다시 받아간다.

```vbnet
GET /framework-ab12cd34.js   → 304 Not Modified
GET /lib-ef56gh78.js         → 304 Not Modified
GET /main-mn34op56.js        → 304 Not Modified
GET /index-lm12no34.js       → 200 OK (해시 변경)
```

### 트리쉐이킹

트리쉐이킹이란, 번들링 시 실제로 사용되지 않는(dead code) JavaScript 코드를 제거하는 최적화 기법을 말한다. 예를 들어, lodash 라이브러리에서 debounce 하나만 쓰고 있는데 전체를 import 한다면 사용하지 않는 수십개의 함수도 같이 번들에 포함된다. 트리쉐이킹은 이러한 불필요한 코드를 제거해서 번들을 작고 빠르게 만드는 전략이다.

```jsx
// BAD (전체 임포트 → 트리쉐이킹 비효율)
import _ from 'lodash';

// GOOD (부분 임포트 → 트리쉐이킹에 유리)
import debounce from 'lodash/debounce';
```

Wepback이나 Next.js가 자동으로 트리 쉐이킹을 하려면 아래 조건이 필요하다.

| 조건 | 설명 |
| --- | --- |
| ✅ ESModules 사용 | `import`/`export` 형식이어야 함 (CJS `require`는 안 됨) |
| ✅ `mode: 'production'` | Webpack이 dead code 제거를 수행하는 최적화 모드 |
| ✅ `sideEffects: false` | `package.json`에 명시해야 안전하게 삭제 가능 |

```json
// package.json 예시
{
  "sideEffects": false
}
```

- `false`: 모든 파일은 사이드 이펙트가 없다고 간주 → 트리 쉐이킹 가능
- 배열도 가능: `["*.css"]` 등 → CSS는 유지, JS만 제거

그렇다면, Next.js에서 트리쉐이킹은 어떻게 작동하는걸까?

Next.js는 아무 설정 없이도 트리 쉐이킹을 기본 지원하는데, 이유는 아래와 같다.

1. SWC(Speedy Web Compiler) 기반 빌드
    - Next.js는 Babel 대신 **Rust 기반의 SWC 컴파일러**를 사용
    - `swcMinify: true` 설정으로 **dead code 제거, 압축, 리네이밍까지 자동 수행**
    - `next.config.js`에서 따로 설정 안 해도 기본 활성화됨
2. ESModules을 사용한 코드 분석
    - Next.js는 모든 내부 번들링을 ESM 기반으로 수행
    - 따라서 Webpack이 모듈 사이의 사용 여부를 정확하게 파악 가능
3. 라이브러리 최적화 지원(modularizeImports)
    - MUI, lodash 등 일부 라이브러리는 불필요한 코드가 많이 포함될 수 있어
    - Next.js는 자동으로 `modularizeImports` 설정을 적용하거나 수동 커스터마이징 가능

```jsx
// next.config.js 예시
modularizeImports: {
  lodash: {
    transform: 'lodash/{{member}}',
  },
}
→ 사용한 것만 번들에 포함되도록 함
```

이처럼 Next.js는 기본적으로 **아주 정교한 Webpack 최적화 설정을 내장하고 있으며**,

`splitChunks`, `runtimeChunk`, `contenthash` 기반 캐싱 전략 등을 통해 성능과 네트워크 효율을 자동으로 극대화해준다.

따라서 대부분의 경우, **개발자가 별도로 코드 스플리팅이나 트리 쉐이킹을 설정할 필요가 없을 정도로 잘 구성되어 있다.**
