## Entry & Output

entry는 번들링의 시작점.
entry는 Webpack이 모듈 그래프(한 파일로 시작되서 import로 엮여있는 파일들을 하나로 묶은)를 만들기 위해 처음으로 읽는 파일

멀티 엔트리 포인트 설정
이렇게 하면 여러개의 번들파일(js)로 묶여짐

```js
// webpack.config.js
module.exports = {
  entry: {
    popup: './src/popup.tsx',
    background: './src/background.ts',
    content: './src/content.ts',
  },
  output: {
    filename: '[name].bundle.js', // popup.bundle.js, background.bundle.js 등으로 생성됨
    path: path.resolve(__dirname, 'dist'), // __dirname 현재 webpack.config.js의 절대 경로를 의미함 /  path.resolve는 파일 경로들을 합쳐서 하나의 절대경로를 만들어주는 함수 /
    // webpack.config.js가 /home/user/project/webpack.config.js라면 ==> path.resolve(__dirname, 'dist') === '/home/user/project/dist'
    clean: true, // 이전 빌드 파일 삭제 (옵션)
  },
  // 나머지 설정 (loader, plugin 등)
};

// 혹은

module.exports = {
  entry: ['./src/file_1.js', './src/file_2.js'],
  output: {
    filename: 'bundle.js',
  },
};
// 파일 2개를 1개로 묶을 수 있음(두 파일이 서로 디펜던시가 안걸려있을때 사용)
```

## Loaders

웹팩은 자바스크립트, json형태의 파일만 처리가 가능하기 때문에 타입스크립트,svg와 같은 파일을 처리
하기 위해서 사용하는 모듈로 이해하면 된다.

.ts/.tsx → TypeScript 트랜스파일

.css → JS 안에서 import 가능

.svg → React 컴포넌트로 변환

file-loader → 이미지 경로 처리

```js
...
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: {
          loader: 'ts-loader',
          options: {
            reportFiles: ['src/**/*.{ts,tsx}'],
          },
        },
        exclude: /node_modules/,
      },
      {
        test: /\.css$/i,
        use: ['style-loader', 'css-loader'],
      },
      {
        test: /\.(png|jpe?g|gif)$/i,
        loader: 'file-loader',
      },
      {
        test: /\.svg$/,
        use: ['@svgr/webpack'],
      },
      {
        test: /\.m?js/,
        resolve: {
          fullySpecified: false,
        },
      },
    ],
  },
```

그 중 하나의 옵션값을 살펴보자

### 🔧 전체 설정

```js
{
  test: /\.tsx?$/,
  use: {
    loader: 'ts-loader',
    options: {
      reportFiles: ['src/**/*.{ts,tsx}'],
    },
  },
  exclude: /node_modules/,
}
```

이 코드가 의미하는 바는 다음과 같다

`.ts` 또는 `.tsx` 파일들을 `ts-loader`를 사용해 처리한다.

---

---

### ✅ 각 항목 설명

#### `test: /\.tsx?$/`

- 정규식 `/\.tsx?$/`는 `.ts` 또는 `.tsx`로 끝나는 파일을 대상으로 로더를 적용시킨다.
- 즉, TypeScript 파일(`.ts`, `.tsx`)만 처리 대상.

#### `use: { loader: 'ts-loader', ... }`

- `ts-loader`는 TypeScript 코드를 JavaScript로 변환해주는 Webpack용 로더.
- 내부적으로 `tsc` (TypeScript 컴파일러)를 사용.

#### `options: { reportFiles: ['src/**/*.{ts,tsx}'] }`

- 이 옵션은 `ts-loader`가 **어떤 파일들만 포함시킬지**를 제한.
- `reportFiles`는 TypeScript 프로젝트 전체에서 **컴파일하거나 타입 검사에 포함할 파일들만 필터링**하는 역할을 합니다.
- `'src/**/*.{ts,tsx}'`는 `src` 폴더 하위의 `.ts` 또는 `.tsx` 파일들만을 대상으로 합니다.

  - 다른 경로나 테스트 디렉터리 등은 무시됨
  - 예: `test/**/*.ts`는 타입 체크에서 제외

> 이 옵션은 **ts-loader가 tsconfig에서 포함한 모든 파일을 자동으로 처리하는 것을 방지**하고, 명시한 glob 패턴의 파일만 처리하고 싶을 때 유용합니다.

#### `exclude: /node_modules/`

- `node_modules` 폴더는 트랜스파일 대상에서 제외됩니다.
- 일반적으로 써드파티 라이브러리는 이미 컴파일되어 있으므로 제외하는 게 성능에 이롭습니다.

---

### 📝 참고: `reportFiles` 옵션이 필요한 이유

기본적으로 `ts-loader`는 `tsconfig.json`에서 설정한 `include`/`exclude`에 따라 전체 타입 검사를 진행합니다.
하지만 Webpack의 엔트리만 빌드하고 싶을 때, **불필요한 파일까지 타입 체크하지 않도록 제한**할 수 있습니다.

예를 들어:

- `tsconfig.json`에는 전체 프로젝트를 포함하고 있어도
- 실제 Webpack은 `src/` 아래 코드만 번들링할 경우
- `reportFiles`를 통해 타입 검사 범위를 최소화할 수 있음

---

### ✅ 정리

| 항목                      | 설명                                 |
| ------------------------- | ------------------------------------ |
| `test: /\.tsx?$/`         | `.ts`, `.tsx` 파일만 처리            |
| `loader: 'ts-loader'`     | TypeScript를 Webpack에서 컴파일      |
| `options.reportFiles`     | 타입 검사/트랜스파일 대상 파일 제한  |
| `exclude: /node_modules/` | 외부 라이브러리는 무시하여 성능 향상 |

---

필요하면 `tsconfig.json`과 어떻게 연동되는지도 함께 설명드릴 수 있습니다.

## plugins

번들할때 기능 추가할때 사용

폴리필, 환경변수 주입같은거에서 사용함.

이러면 js파일에서 환경변수 접근 가능

```js
plugins: [new Dotenv({ path: './.env.chrome.development' })],
```

## resolve

모듈을 어떻게 해석하고 찾을지에 대한 규칙을 정의하는 설정값
import 경로를 깔끔하게 줄이고, 확장자 생략할때 사용.

```js
  resolve: {
    extensions: ['.ts', '.tsx', '.js'], // import할때 확장자 안써도 되게끔해줌 (ex. import MyComponent from './MyComponent';) tsx → .ts → .js 파일을 찾는다
    plugins: [new TsconfigPathsPlugin()], // TsconfigPathsPlugin은 Webpack이 TypeScript의 tsconfig.json에 정의된 paths 별칭을 인식할 수 있게 해주는 플러그인
    fallback: {
      stream: 'stream-browserify',
      assert: 'assert',
      os: 'os-browserify',
      url: 'url',
      http: 'stream-http',
      https: 'https-browserify',
      crypto: 'crypto-browserify',
    },
  },
```

### 모듈 import 경로 설정하기 - TsconfigPathsPlugin

Webpack은 tsconfig.json의 paths 설정을 기본적으로 알지 못하니까
Webpack에서도 tsconfig와 같이 똑같은 경로를 인식하려면 TsconfigPathsPlugin을 써야 합니다.

```js
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "src",
    "paths": {
      "@components/*": ["components/*"],
      "@utils/*": ["utils/*"]
    }
  }
}
```

이렇게 하면 TypeScript에서 다음과 같은 import가 가능

```ts
import MyButton from '@components/MyButton';
```

위 코드엔 `TsconfigPathsPlugin`를 썻지만 alias라는 옵션을 사용해도됨

alias
긴 상대 경로 대신 짧은 경로 별칭 설정

```js
alias: {
  '@utils': path.resolve(__dirname, 'src/utils'),
  '@assets': path.resolve(__dirname, 'src/assets'),
}
```

```ts
// Before
import Logo from '../../../assets/logo.svg';

// After
import Logo from '@assets/logo.svg';
```

mainFiles
디렉토리 import 시 기본으로 찾을 파일 이름 지정

```ts
resolve: {
  mainFiles: ['index', 'main'];
}
```

// 디렉토리만 써도 index.tsx나 main.tsx 등을 자동 인식

```ts
import MyPage from './pages/mypage';
```

fallback
Node.js 내장 모듈을 사용할 때 대체 모듈을 지정
노드 js에서는 사용가능하지만 브라우저환경에서는 지원하지 않는 모듈을 사용할때 폴리필용으로 사용

```ts
resolve: {
  fallback: {
    fs: false, // 서버 전용 모듈 무시
    crypto: require.resolve('crypto-browserify') // 브라우저 대체
  }
}
```

주로 브라우저용 번들에서 fs, path, crypto 등의 Node 전용 모듈을 제거하거나 대체할 때 사용

## 모드설정

모드 옵션을 통해서 개발, 프로덕션 환경을 나눠서 개발이 가능.
보통은 환경에 따라서 소스맵등의 활성 여부 차이를 두거나 할때 사용

사용 예

```js
import i18n from 'i18next';

import enTranslation from './en/translation.json';
import koTranslation from './ko/translation.json';

// eslint-disable-next-line @typescript-eslint/no-floating-promises
i18n.init({
  resources: {
    ko: { translation: koTranslation },
    en: { translation: enTranslation },
  },
  defaultNS: 'translation',
  fallbackLng: 'en',
  // 이렇게 사용함.
  debug: process.env.RUN_MODE === 'development',
  interpolation: {
    escapeValue: false,
  },
});

export default i18n;
```

## 사용해봄직한 패키지

### webpack-merge

const { merge } = require('webpack-merge');
웹팩 설정들을 합쳐주는 라이브러리

### dotenv-webpack

.env 파일에 정의된 환경변수를 Webpack 빌드 시점에 JS 코드 내에서 사용할 수 있게 해주는 플러그인입니다.
`.env` 파일의 내용을 `process.env`로 코드에 주입해줌

사용예

```js
const { merge } = require('webpack-merge');
const common = require('./webpack.common.chrome.js');
const Dotenv = require('dotenv-webpack');
const path = require('path');

// common의 설정값들을 복사해서 사용
module.exports = merge(common, {
  mode: 'production',
  plugins: [new Dotenv({ path: './.env.chrome' })],
  output: {
    path: path.join(__dirname, '../build/chrome/prod/js'),
    filename: '[name].js',
  },
});
```

```js
const { merge } = require('webpack-merge');
const common = require('./webpack.common.chrome.js');
const Dotenv = require('dotenv-webpack');
const path = require('path');

// common의 설정값들을 복사해서 사용
module.exports = merge(common, {
  mode: 'production',
  plugins: [new Dotenv({ path: './.env.chrome' })],
  output: {
    path: path.join(__dirname, '../build/chrome/prod/js'),
    filename: '[name].js',
  },
});
```

```js
export async function getAllSessionStorage(): Promise<ExtensionSessionStorage> {
  if (process.env.BROWSER === 'chrome') {
    return new Promise((res, rej) => {
      chrome.storage.session.get(null, (items) => {
        if (chrome.runtime.lastError) {
          rej(chrome.runtime.lastError);
        }
        res(items as ExtensionSessionStorage);
      });
    });
  }
  const localStorage = await browser.storage.session.get();

  return localStorage as ExtensionSessionStorage;
}
```

## 최적화 옵션들

### 1. 코드 스플리팅

큰 하나의 js파일을 여러개로 쪼개는 행위
왜 쪼갬?

1. 브라우저에서 js파일을 다운 받을때 메인 번들이 1~2MB가 넘으면 네트워크 상태가 안 좋은 환경에서는 로딩에 꽤 시간이 걸림 -> 그래서 최대한 한방에 다운로드 받으려고 코드를 쪼개는것

main.js (앱의 핵심 로직)
vendors~react.js (React 라이브러리)
featureA.js (사용자가 특정 버튼 클릭 시 로드하는 기능)

이렇게 파일을 쪼갠다고 하면

사용자는 처음에 main.js와 vendors~react.js만 다운로드 →
featureA가 필요할 때 동적으로 featureA.js를 불러옴.

- 자동 청크화

```ts
optimization: {
  splitChunks: {
    chunks: 'all',
  },
}
```

위 옵션대로 쓰면 청크화될 수 있는대로 다 쪼갠다는 의미, 자동 청크화

- `node_modules` 같은 공통 모듈을 분리하여 캐시 효율을 높임

- 동적 import()

```ts
import('./MyComponent').then(({ default: MyComponent }) => {
  // MyComponent를 여기서만 로딩
});
```

코드 일부를 함수 내에서 동적으로 import()하면, Webpack이 이 부분을 별도 청크로 분리

React에서는 React.lazy와 Suspense로 사용함

```tsx
const MyComponent = React.lazy(() => import('./MyComponent'));
```

주의점

자동청크화로 쪼갰을때 파일이 너무 쪼개지면

쪼갠 파일들 마다 다 네트워크 요청을 날려야하기떄문에 오버헤드가 발생할 수 있으니 적당히 쪼개는것도 중요함.

파일이 너무 많으면 각 파일마다 이런 과정을 거쳐야하는데 굳이? 이런 느낌

- TCP 연결 및 핸드셰이크
- TLS 연결(HTTPS라면 SSL 인증 절차 포함)
- HTTP 헤더 전송
- 브라우저의 요청 큐/순서 대기
- 응답 헤더 및 메타 처리
- 캐시 확인 처리

이런걸 고려한다면 이렇게 쓸 수 있음

```js
splitChunks: {
  chunks: 'all',
  minSize: 30000, // 최소 30KB 이상일 때만 분리
  cacheGroups: {
    defaultVendors: {
      test: /[\\/]node_modules[\\/]/, // 노드 모듈의 파일들만 대상
      name: 'vendors', //  청크 이름
      chunks: 'all',
    },
  },
}
```

예전에 썻던 코드

```js
  optimization: {
    splitChunks: {
      name: 'vendor', // 청크 이름을 벤더로 한다
      chunks(chunk) { // chunks는 엔트리 청크에 대해 스플리팅을 적용시킬지 말지를 판단하는 함수
      // 엔트리 청크란 앞에서
        //     entry: {
        //     popup: './src/popup.tsx',
        //     background: './src/background.ts',
        //     content: './src/content.ts',
        //   },
        // 이렇게 해놨을때 popup.js, background.js를 의미한다
        const notChunks = ['background', 'injectScript'];
        return !notChunks.includes(chunk.name);
        // 여기서는 background.js청크랑 injectScript.js청크에는 코드 스플리팅을 적용안시킨다는거임.
        // 백그라운드 스크립트는 타 스크립트랑 독립적으로(서비스워커에서 실행되어서 실행영역이 아예 다름) 실행되기 때문에 안나눔
        // 인젝트 스크립트는 웹페이지 돔에 삽입되는 코드라서 얘도 독립적으로 온전히 구성되어야해서 코드 스플리팅 제외시킴
      },
    },
  },
```

### 2. 캐시 최적화

```ts
output: {
  filename: '[name].[contenthash].js',
}
```

- 파일 이름에 `contenthash`를 붙여 **내용이 바뀌지 않으면 브라우저 캐시 유지**

### 3. Tree Shaking

- `mode: 'production'`일 때 자동 활성화
- 사용하지 않는 코드 제거
- ES Module을 쓰는 것이 핵심 (`import/export`)

### 4. 압축 (Minify)

```ts
mode: 'production';
// or
optimization: {
  minimize: true;
}
```

- JS/CSS 압축은 기본 제공됨 (`TerserPlugin`, `css-minimizer-webpack-plugin`)

### 5. 이미지 최적화 (선택)

```bash
npm install image-webpack-loader --save-dev
```

```ts
{
  test: /\.(png|jpe?g|gif)$/i,
  use: [
    'file-loader',
    {
      loader: 'image-webpack-loader',
      options: {
        mozjpeg: { progressive: true },
        optipng: { enabled: false },
        pngquant: { quality: [0.65, 0.90], speed: 4 },
      },
    },
  ],
}
```

---

추가 설명

---

## ✅ 1. 코드 스플리팅 (Code Splitting)

```ts
optimization: {
  splitChunks: {
    chunks: 'all',
  },
}
```

### 💡 목적

공통 코드(특히 `node_modules`)를 **별도 파일로 분리**해서 중복 없이 공유하고, 브라우저 캐시 효율을 높이는 전략.

### 🔍 예시

여러 페이지/컴포넌트에서 `lodash`, `axios` 등을 각각 import할 때 →
Webpack이 이를 하나의 `vendors~.js` 같은 chunk로 분리해서 각 페이지에서 재사용 가능.

### 📦 출력 예시

- `vendors~lodash.abc123.js` ← 공통 모듈
- `page1.123abc.js` ← 페이지 전용 코드
- `page2.456def.js` ← 다른 페이지 전용 코드

이렇게 되면, 사용자가 페이지 간 이동 시 공통 코드는 캐시로 유지되고, 필요한 페이지만 다운로드하므로 **속도와 효율이 개선됩니다**.

---

## ✅ 2. 캐시 최적화 (Cache Optimization)

```ts
output: {
  filename: '[name].[contenthash].js',
}
```

### 💡 목적

**파일의 내용이 바뀌지 않으면 파일 이름도 안 바뀌게** 해서 브라우저 캐시를 적극 활용하는 전략.

### 🔥 왜 중요한가?

- 기본 설정(`bundle.js`) → 새로 배포하면 항상 파일명이 같음 → 매번 새로 다운로드됨 (비효율적)
- `contenthash` → 내용이 변해야 파일명이 바뀜 → 바뀐 파일만 다운로드됨 (효율적)

### 📦 출력 예시

- `/dist/main.21af0c9a.js`
- `/dist/vendors~react.89dd93f0.js`

### 💬 보통 사용되는 방식

- `contenthash`: JS, CSS에 사용됨 (내용 기반 캐싱)
- `chunkhash`: Chunk 단위 해시 (과거 방식)
- `hash`: 전체 빌드 해시 (모든 파일이 동일 해시 → 비추천)

---

## ✅ 3. Tree Shaking

### 💡 목적

**사용하지 않는 import 코드를 제거**해서 번들 크기를 줄이는 기능.

### 🧠 전제 조건

- `mode: 'production'`
- ES Modules(`import/export`) 사용 (es6를 사용한다는 의미임)
- side-effect 없는 코드 구조

사이드 이펙트가 없다는건 전역 변수 변경, DOM 조작, 로그 출력와 같이 실행되는 함수이외의 무언가가 또 실행되는거

```js
export function add(a, b) {
  return a + b;
}
```

위 함수는 호출하지 않으면 아무 일도 안 일어나므로 부작용 없는 코드.

반면,

```js
console.log('실행됨');
```

이런 건 사이드 이펙트가 있어서 제거하면 안 됩니다.

- 라이브러리로 가져다가 쓰는 코드도 셰이킹대상임
- import { funcA } from 'lib' 할 때, funcA만 쓰고 나머지 안 쓰면 그 부분 코드를 제거
- 단 esm으로 작성되어야하며 커먼js는 작동안함.

### 🛑 주의 사항

패키지가 제대로 tree shaking 안 되면 용량만 커짐
→ package.json에 `sideEffects: false` 설정하거나, 개별 파일에 주석(`/*#__PURE__*/`) 처리

```json
// package.json
{
  "sideEffects": false // 이렇게 쓰면 사이드 이펙트없다고 단언하는거
}
```

### ✅ 예시

```ts
// lodash 전체 불러오기 (Bad)
import _ from 'lodash';

// 필요한 것만 (Good)
import debounce from 'lodash/debounce';
```

---

## ✅ 4. 압축 (Minification)

```ts
mode: 'production';
// 또는
optimization: {
  minimize: true;
}
```

### 💡 목적

**JS/CSS의 공백, 주석, 변수명 등을 제거**해서 번들 크기 최소화.

### 기본 사용 플러그인

- JS: `TerserPlugin` (기본 내장)
- CSS: `css-minimizer-webpack-plugin`

### ✨ 결과

- 가독성이 없는 단일 줄 코드로 압축됨
- 변수를 짧은 문자로 바꿔 크기 절감

```ts
const longVariableName = 5;
→ var a = 5;
```

---

## ✅ 5. 이미지 최적화

```bash
npm install image-webpack-loader --save-dev
```

```ts
{
  test: /\.(png|jpe?g|gif)$/i,
  use: [
    'file-loader',
    {
      loader: 'image-webpack-loader',
      options: {
        mozjpeg: { progressive: true },
        optipng: { enabled: false },
        pngquant: { quality: [0.65, 0.9], speed: 4 },
      },
    },
  ],
}
```

### 💡 목적

이미지 용량을 줄여 초기 로딩 속도를 빠르게 함.
용량이 큰 PNG, JPEG 등을 자동으로 최적화해줌.

### 플러그인 기능

| 이름       | 역할                             |
| ---------- | -------------------------------- |
| `mozjpeg`  | JPEG 압축                        |
| `optipng`  | PNG 압축 (disabled 위 예시에서)  |
| `pngquant` | PNG 압축 퀄리티/속도 조절        |
| `svgo`     | SVG 최적화 지원 (별도 설정 필요) |

### 📦 대체 방법

- 이미지 CDN 사용
- 빌드 시 WebP 변환
- `vite-imagetools`, `next/image` 같은 고급 이미지 최적화 도구

---

## 🧠 마무리 요약

| 항목          | 설명                        | 효과               |
| ------------- | --------------------------- | ------------------ |
| 코드 스플리팅 | 공통 코드를 분리            | 초기 로딩 최적화   |
| 캐시 최적화   | `contenthash`로 파일명 고정 | 브라우저 캐시 활용 |
| Tree Shaking  | 사용하지 않는 코드 제거     | 번들 크기 감소     |
| 압축          | JS/CSS 압축                 | 파일 사이즈 감소   |
| 이미지 최적화 | 이미지 자동 압축            | 로딩 속도 향상     |

---

원하시면 이걸 기반으로 블로그용 마크다운 포맷으로 깔끔하게 정리해드릴 수도 있어요.

## 기타 팁

- **`devtool: 'source-map'`** → 디버깅용 소스맵
- **`resolve.extensions`** 설정 → import 시 확장자 생략 가능
- **멀티 페이지 앱**은 `HtmlWebpackPlugin`을 여러 개 사용하면 처리 가능
