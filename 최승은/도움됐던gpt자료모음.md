## 1. **Chunk Split 전략 심화 (`splitChunks`)**

### 💡 왜 중요한가?

* 번들 개수가 너무 많아지면 오히려 HTTP/2 이점 사라짐 → 분리 기준이 핵심
“번들을 나누면 좋은 거 아닌가요?” → 대부분 그렇게 생각하지만, 무작정 많이 나누면 오히려 성능이 떨어질 수 있어요.
HTTP/2는 기존 HTTP/1.1과 달리 단일 연결에서 병렬로 요청을 처리할 수 있음 → 이 때문에 '청크를 많이 나누면 브라우저가 동시에 받겠지!'라고 착각할 수 있어요.

하지만 실제로는 분할 개수와 용량, 요청 수, 서버 세팅이 모두 맞물려서 다음과 같은 부작용이 생깁니다:
좋은 질문이에요!
“**번들을 나누면 좋은 거 아닌가요?**” → 대부분 그렇게 생각하지만, ***무작정 많이 나누면** 오히려 성능이 떨어질 수 있어요.*
특히 HTTP/2 환경에서 이게 왜 문제가 되는지를 실무적으로 이해하면, `splitChunks`나 `manualChunks`를 훨씬 똑똑하게 쓸 수 있게 됩니다.

## 🔍 HTTP/2의 특징부터 이해하자

HTTP/2는 기존 HTTP/1.1과 달리 **단일 연결에서 병렬로 요청을 처리할 수 있음** → 이 때문에 '청크를 많이 나누면 브라우저가 동시에 받겠지!'라고 착각할 수 있어요.

하지만 실제로는 **분할 개수와 용량, 요청 수, 서버 세팅**이 모두 맞물려서 다음과 같은 부작용이 생깁니다:

## ⚠️ 번들을 너무 많이 나눴을 때 발생하는 문제들

### 1. **서버 푸시/프리페치 미스**

* HTTP/2 Server Push는 성능 최적화를 위해 미리 리소스를 푸시해주는데, 너무 많은 리소스를 푸시하면 클라이언트가 오히려 **필요하지 않은 걸 먼저 받게 될 수 있음**
* 브라우저가 cache hit miss를 잘못 판단해서 다시 받는 경우도 있음

### 2. **브라우저가 너무 많은 JS 요청을 동시에 처리하면 오히려 병목 발생**

* 실제로는 네트워크보다 **JS 해석/실행 시간**이 더 오래 걸림
* 작은 파일 100개를 동시에 받아도 **main thread가 blocking됨**
  → 병렬 다운로드는 되지만, 실행 순서가 꼬이면 **렌더링 지연 발생**

### 3. **헤더 크기 증가 → 성능 저하**

* HTTP/2는 여러 리소스를 한 연결로 요청하지만, 요청이 많아지면 header table 압력이 커짐
* 특히 모바일에서 성능 저하 체감이 큼

## ✅ 그래서 “적절한” 청크 분리가 중요

### 📌 좋은 분리 기준 예시

| 기준                  | 설명                                        |
| ------------------- | ----------------------------------------- |
| `route-level` 청크    | 페이지 단위로 분리 (ex. 대시보드, 마이페이지)              |
| `vendor` 청크         | react, lodash 등 제3자 라이브러리 분리              |
| `heavy module` 청크   | 대형 차트, rich text editor 등 무거운 컴포넌트만 별도 로딩 |
| `기본 청크 수는 5~10개 이하` | 너무 많으면 초기 병렬 처리 이득이 사라짐                   |

## 💡 Webpack에서의 실제 전략 예시

```js
optimization: {
  splitChunks: {
    chunks: 'all',
    minSize: 30000, // 30KB 이상만 분리
    maxAsyncRequests: 5, // 동시에 로딩할 async 청크 수 제한
    maxInitialRequests: 3, // 초기 로딩 시 청크 요청 제한
    cacheGroups: {
      vendors: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendor',
        priority: -10,
      },
    },
  },
}
```

이렇게 하면,

* 초기 렌더 시 3개 이하로만 로딩됨
* 동적 import로 나중에 로드할 청크도 5개까지만 제한
* vendor 파일은 자동 분리

### 📌 실습 아이디어

* `splitChunks.cacheGroups`로 `vendors`, `common`, `styles` 따로 쪼개기
* 각 그룹마다 `test`, `priority`, `minSize`, `minChunks` 조건 조절
* Chrome DevTools Network 탭으로 분할 전략 검증

## 🔚 요약

* **HTTP/2가 있다고 해서 무조건 청크를 많이 나누는 게 이득은 아님**
* JS는 다운로드보다 **해석·실행 비용**이 더 큼
* **렌더링 차단을 최소화하고**, 필요한 시점에만 불러오는 구조가 가장 효율적
* splitChunks 설정에서 `minSize`, `maxInitialRequests`, `manualChunks()` 등을 적극 활용해야 함

---

## 2. **Dynamic Import 시 prefetch, preload 제어**

### 💡 왜 중요한가?

* 너무 늦게 로딩되면 UX 저하, 너무 빨리 prefetch되면 초기 로딩 느려짐
prefetch, preload를 사용하는 이유는 사용자가 클릭하기 전에 JS 청크를 미리 받아놓고자 하는 건데,
잘못 쓰면 초기 성능이 오히려 나빠집니다.

## ✅ 우선 용도부터 정리

| 속성         | 용도                           | 동작 시점                 | 브라우저 우선순위                   |
| ---------- | ---------------------------- | --------------------- | --------------------------- |
| `preload`  | **곧바로 필요할 리소스**를 먼저 불러옴      | `<script>` 로딩 시작과 동시에 | 매우 높음 (렌더링 blocking 가능성 있음) |
| `prefetch` | **나중에 쓸 리소스**를 유휴 시간에 미리 받아둠 | 브라우저가 한가할 때           | 매우 낮음 (lazy)                |

## ⚠️ 문제가 되는 상황 예시

### ❌ 1. **무분별한 prefetch → 초기 로딩 느려짐**

* 페이지에 들어가자마자 관련 없는 페이지용 청크들까지 `prefetch`로 한 번에 요청됨
* 모바일이나 느린 네트워크에서는 JS 다운로드만으로도 **main thread 병목** 발생

📎 *예시*

```js
import(/* webpackPrefetch: true */ './ChartPage'); // 메인 페이지에서 필요 없음
```

➡️ **결과:** 사용자가 차트 페이지에 가지도 않았는데 200KB짜리 모듈이 백그라운드에서 내려받아지고, main thread는 느려지고, 실제 main 화면 렌더링이 늦어짐

### ❌ 2. **preload 남발 → 렌더링 차단**

* preload는 반드시 즉시 사용할 리소스에만 써야 함
* preload된 리소스는 우선순위가 높아서 **다른 중요한 CSS/JS보다 먼저 다운로드됨**

📎 *예시*

```js
import(/* webpackPreload: true */ './VideoPlayer'); // 페이지 진입 직후엔 필요 없음
```

➡️ **결과:** 메인 콘텐츠보다 VideoPlayer가 먼저 다운로드됨 → 화면이 늦게 뜸

## 💡 그러면 어떻게 쓰는 게 적절할까?

| 케이스                             | 추천 방식                      |
| ------------------------------- | -------------------------- |
| 사용자가 곧 클릭할 가능성이 높은 버튼/메뉴        | `prefetch`                 |
| route-level에서 진입 직후 바로 필요한 컴포넌트 | `preload`                  |
| 사용 여부가 불확실한 페이지                 | ❌ 아무 처리도 하지 않기 (진입 시점에 로딩) |

## ✅ Webpack 기준 사용 예시

```js
// prefetch → 사용자가 자주 가는 메뉴지만, 지금 당장은 필요 없음
import(/* webpackChunkName: "settings", webpackPrefetch: true */ './SettingsPage');

// preload → 라우팅 직후 바로 보여줄 컴포넌트
import(/* webpackChunkName: "chart", webpackPreload: true */ './ChartView');
```

## 🧪 실험할 수 있는 비교 포인트

| 실험 내용                                  | 관찰 위치                             |
| -------------------------------------- | --------------------------------- |
| prefetch → Network 탭에서 요청 우선순위가 낮게 뜨는지 | Chrome DevTools → Priority: "Low" |
| preload → 실제 렌더링 속도가 빨라졌는지             | Chrome DevTools → Performance 탭   |
| 두 옵션 제거 후와 비교해서 **렌더링 차이** 확인          | LCP, FCP 지표 비교 or Lighthouse      |

## ✅ 결론 정리

* `prefetch`는 “이 페이지 다음에 뭐 누를지 예상할 수 있을 때” 쓰는 것
* `preload`는 “지금 페이지가 **렌더링 완료되기 전에** 꼭 필요한 리소스일 때”만 쓴다
* 무작정 붙이지 말고, **사용자 행동 동선과 타이밍을 함께 고려해야 한다**


### 📌 실습 포인트

```js
import(/* webpackChunkName: "chart", webpackPrefetch: true */ './Chart.js');
```

* `webpackPrefetch`: 유휴 시간에 미리 불러옴
* `webpackPreload`: 현재 바로 필요할 가능성 있는 경우

---

## 3. **Runtime Chunk 분리 (`runtimeChunk: 'single'`)**

### 💡 왜 중요한가?

* webpack runtime 코드가 entry마다 중복 포함되면 캐시 효율 떨어짐

```js
optimization: {
  runtimeChunk: 'single',
}
```

* 하나로 묶으면 코드 변경 없는 한, long-term cache 활용 가능

## 🔍 먼저, Webpack의 runtime이란?

* Webpack은 JS 모듈을 번들로 묶은 다음, 이를 실행할 수 있도록 **자신만의 실행 코드(runtime)** 를 같이 포함합니다.
* 이 runtime은 다음과 같은 일을 합니다:

| 기능                      | 설명                     |
| ----------------------- | ---------------------- |
| `__webpack_require__()` | 각 모듈을 ID로 로드하고 실행함     |
| `__webpack_modules__`   | 전체 모듈 맵 관리             |
| chunk id 해시 추적          | 청크 파일들의 매핑 정보 관리       |
| HMR 지원                  | 모듈 교체 시 동작 로직 포함 (개발용) |

> 📌 즉, 번들 자체를 실행하는 로더 역할을 하는 **부트스트랩 코드**

## ⚠️ 그런데 entry마다 runtime이 포함되면?

### 📎 예시 상황

```js
entry: {
  main: './src/index.js',
  admin: './src/admin.js'
}
```

빌드 결과:

* `main.[hash].js`
* `admin.[hash].js`
  → 각각에 Webpack runtime 코드가 포함됨

### ❌ 문제점 1: **중복 코드가 발생한다**

* main과 admin은 서로 다른 entry지만, 둘 다 Webpack runtime을 가지고 있음
* 동일한 runtime인데, 파일명이 다르고 매번 해시도 달라짐 → 브라우저 캐시 사용 불가

### ❌ 문제점 2: **작은 변경에도 캐시가 깨진다**

* 예를 들어 main entry만 변경되었더라도,

  * 그로 인해 `main.[hash].js`의 해시가 바뀜
  * runtime 코드도 함께 바뀐 것으로 간주되어 **다시 다운로드됨**

➡️ 실제 변경된 건 main 코드 일부인데, 전체 runtime까지 재요청됨 → 캐시 비효율 + 네트워크 낭비

## ✅ 해결 방법: `runtimeChunk: 'single'`

### 설정 방법

```js
optimization: {
  runtimeChunk: 'single',
}
```

### 효과

* runtime이 **하나의 독립 청크(`runtime~main.js`)로 분리**됨
* 모든 entry가 이 runtime 청크를 **공통으로 사용**함
* → runtime이 바뀌지 않으면 캐시 그대로 유지 가능

### 결과

빌드 결과:

* `runtime~main.[hash].js` ✅ (공통)
* `main.[hash].js` ✅
* `admin.[hash].js` ✅

→ 이렇게 되면 작은 변경이 있더라도 **runtime은 그대로 캐싱되고**, 필요한 파일만 다시 받게 됨


## 📌 실무에서는 어떻게 적용할까?

| 상황                               | 적용 여부                                               |
| -------------------------------- | --------------------------------------------------- |
| 멀티 엔트리 구조 (admin, dashboard 등)   | 무조건 적용 권장                                           |
| 싱글 페이지 앱 (SPA)                   | runtimeChunk 없어도 큰 차이는 없음 (단, caching 최적화엔 여전히 도움됨) |
| 코드 스플리팅이 많고, 동적 import를 많이 쓰는 경우 | 적용 시, 캐시 관리가 훨씬 쉬움                                  |

## ✅ 요약 정리

* Webpack의 runtime 코드는 번들을 실행하기 위한 핵심 부트스트랩 로직이다
* 멀티 엔트리나 동적 import가 많은 경우, 이 runtime 코드가 여러 청크에 중복되면 캐시 효율이 떨어진다
* `runtimeChunk: 'single'` 설정을 통해 runtime을 분리하면 **중복 제거 + 캐시 효율**을 동시에 잡을 수 있다.

---

## 4. **Webpack의 Module Federation (모듈 연합)**
> 여러 개의 서로 다른 Webpack 빌드 결과물(앱)들이 런타임에 모듈을 공유하고, 서로 불러올 수 있도록 하는 기능
> 즉, A 앱에서 B 앱의 컴포넌트나 모듈을 직접 불러와서 실행할 수 있음 컴파일 타임이 아닌 런타임에 이뤄짐, 마치 여러 앱이 하나처럼 동작

Webpack Module Federation(모듈 연합)은 Webpack 5에서 도입된 기능으로, 마이크로 프론트엔드(Micro Frontend) 구조를 실현할 수 있도록 해주는 굉장히 강력한 기능입니다.
실무에서도 사일로(silo) 조직 구조에서 점진적인 리팩토링, 팀 간 독립 배포 등에 사용되고 있고, Next.js, CRA 기반 프로젝트에서도 적용된 사례가 많습니다.

### 💡 왜 중요한가?

* **Micro Frontend** 구조에서 서로 다른 앱끼리 번들 공유 가능
* 팀 간 배포 독립성 확보 → 대규모 조직에서 매우 중요

### 📌 실습 예시

* 호스트 앱 + 리모트 앱 두 개 구성
* `remoteEntry.js`로 각자 빌드된 모듈 불러오기

---

## 5. **Vite에서 Rollup의 `manualChunks()` 고급 분리 전략**

### 💡 왜 중요한가?

* 자동 분리 외에도, 상황별로 코드 그룹핑 필요 (예: 대형 라이브러리, 유틸 그룹 등)

```ts
manualChunks(id) {
  if (id.includes('node_modules')) {
    if (id.includes('react')) return 'vendor-react';
    if (id.includes('lodash')) return 'vendor-lodash';
    return 'vendor';
  }
}
```

---

## 6. **Third-party 라이브러리 트리쉐이킹 테스트**

### 💡 왜 중요한가?

* 라이브러리가 ESM 형태라도 실제로 tree-shakable 하지 않을 수 있음

### ✅ 실습 꿀팁

* `import * as _ from 'lodash'` → 전체 import (⚠️ bad)
* `import { debounce } from 'lodash-es'` → tree-shakable
* 분석 도구로 실제 결과 비교

---

## 7. **빌드 사이즈 분석 자동화**

### 💡 왜 중요한가?

* 매 릴리즈마다 번들 사이즈 변화 추적 가능 → CI에서 품질 지표로 사용

### 📌 방법

* `webpack-bundle-analyzer` + `webpack-stats-plugin` 활용
* GitHub Actions에서 PR마다 분석 리포트 생성

---

## 8. **esbuild, SWC, Turbopack 등 차세대 번들러 체험**

### 💡 왜 중요한가?

* webpack의 대체제가 점점 현실로 다가오는 중
* esbuild는 Vite 내부 빌드 엔진이기도 함

### ✅ 실습 포인트

* 같은 프로젝트를 esbuild, swc로 빌드해보기
* 빌드 속도/결과물 비교 → 체감 크고 재밌음

---

## 9. **디버깅을 위한 SourceMap 관리 전략**

### 💡 왜 중요한가?

* 번들된 코드는 디버깅이 어려움 → source map이 필수
* 보안 이슈로 인해 production에서는 비공개 서버에 따로 업로드하는 전략도 사용됨

```js
devtool: 'source-map' // Webpack
build: { sourcemap: true } // Vite
```


