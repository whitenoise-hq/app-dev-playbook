# 01. 프로젝트 초기 세팅

> 새 Expo(React Native) 앱을 만들 때 제일 먼저 하는 세팅. 스택·명령어의 상세 기준은 [expo-ios-playbook.md](./expo-ios-playbook.md) 참조.
> 방침: **iOS(App Store)만 배포. TypeScript. Expo Router.**

---

## 1. 프로젝트 생성

```bash
npx create-expo-app@latest <app-name> --template default   # TypeScript 기본
cd <app-name>
```

- **패키지 매니저는 npm 권장** (Expo/RN 기본값, 평평한 node_modules라 마찰 적음).
  - pnpm을 쓸 거면 반드시 `.npmrc`에 `node-linker=hoisted` 추가(안 그러면 `babel-preset-expo` 해석 실패로 번들이 깨진다).

## 2. 핵심 패키지 설치

버전 호환을 위해 **`npx expo install`** 사용(그냥 `npm install` 아님).

```bash
# 라우팅·상태·백엔드
npx expo install expo-router @tanstack/react-query @supabase/supabase-js
npx expo install @react-native-async-storage/async-storage react-native-url-polyfill base64-arraybuffer

# 이미지·미디어
npx expo install expo-image-picker expo-image-manipulator expo-file-system expo-sharing

# UI·애니메이션·폰트
npx expo install expo-font expo-linear-gradient expo-splash-screen expo-status-bar
npx expo install react-native-reanimated react-native-worklets   # reanimated 4는 worklets 필수
npx expo install react-native-gesture-handler react-native-safe-area-context react-native-screens @expo/vector-icons

# 인증
npx expo install expo-apple-authentication expo-auth-session expo-web-browser
```

## 3. babel.config.js

```js
module.exports = function (api) {
  api.cache(true)
  return {
    presets: ['babel-preset-expo'],
    plugins: ['react-native-worklets/plugin'],   // reanimated 4용. 배열 맨 마지막.
  }
}
```
> npm이면 `babel-preset-expo`가 자동 해석됨. pnpm(비-hoisted)이면 `npx expo install babel-preset-expo`로 직접 의존성 추가 필요.

## 4. 폴더 구조

```
app/
  (auth)/login.tsx
  (tabs)/_layout.tsx      # 탭 네비게이터
  _layout.tsx
components/               # 재사용 UI
  ui/                     # Text·Card·Button·Screen 등 토큰 적용 공용 컴포넌트
lib/                      # supabase 클라이언트, query-client, 유틸
hooks/                    # React Query 커스텀 훅
theme/colors.ts           # 색상 단일 출처 (hex 직접 박지 않기)
types/                    # 공용 타입
supabase/
  functions/              # Edge Functions
    _shared/              # 서버 공용 로직 (계산 등 단일 출처)
  migrations/             # SQL (테이블, RLS)
docs/
```

## 5. 골격 파일

**theme/colors.ts** — 모든 색을 여기서만. 컴포넌트에 hex 직접 금지. NativeWind 안 씀.

**lib/query-client.ts**
```ts
import { QueryClient } from '@tanstack/react-query'
export const queryClient = new QueryClient()
```

**lib/supabase.ts** — [02. Supabase 세팅](./setup-02-supabase.md) 참조.

**.env.example** (플레이스홀더만, 실제 `.env`는 gitignore)
```
EXPO_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

**.gitignore** 에 포함 확인: `.env`, `.env.*`(단 `!.env.example`), `/ios`, `/android`

## 6. app.json 기본 (iOS)

```jsonc
{
  "expo": {
    "ios": {
      "bundleIdentifier": "com.<you>.<app>",   // App Store Connect 앱과 반드시 일치
      "buildNumber": "1",
      "supportsTablet": false,
      "usesAppleSignIn": true,                  // 애플 로그인 쓸 경우
      "infoPlist": { "ITSAppUsesNonExemptEncryption": false }
    }
  }
}
```

## 7. 컨벤션 (요약)

- TypeScript strict, `any` 지양, 공용 타입은 `types/`.
- 컴포넌트 함수형 + named export. 파일명 PascalCase(컴포넌트)/camelCase(유틸·훅).
- 스타일 `StyleSheet.create` + `theme/colors.ts` 단일 출처.
- 아이콘 Ionicons만. **이모지 금지**(기기별 렌더링 불일치).
- 서버 통신은 TanStack Query 훅으로 캡슐화. 컴포넌트에서 직접 fetch/supabase 호출 금지.

---

다음: [02. Supabase 세팅](./setup-02-supabase.md)