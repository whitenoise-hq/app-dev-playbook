# app-dev-playbook

> 새 앱을 만들 때마다 참고하는 개인 공통 문서 모음.
> 스택 기준: **React Native + Expo / TypeScript / Supabase / iOS(App Store)만 배포 / 로컬 Xcode 빌드(EAS 안 씀)**.

## 읽는 순서 (새 앱 세팅)

| # | 문서 | 내용 |
|---|------|------|
| — | [expo-ios-playbook](./setup/expo-ios-playbook.md) | 스택·명령어·함정의 상세 기준 (source of truth) |
| 01 | [프로젝트 초기 세팅](./setup/setup-01-project.md) | Expo 앱 생성 → 패키지 → 폴더구조 → 골격 파일 → app.json |
| 02 | [Supabase 세팅](./setup/setup-02-supabase.md) | 프로젝트/env → 클라이언트 → RLS → Storage → 계정삭제 RPC → Edge Function |
| 03 | [인증 세팅](./setup/setup-03-auth.md) | provider 추상화 → 카카오(웹 OAuth) → 애플(네이티브) → 세션/온보딩 |
| 04 | [iOS 빌드·배포](./setup/setup-04-ios-build.md) | 로컬 Xcode: prebuild → 서명 → Archive → Upload → TestFlight |
| 05 | [App Store 제출](./setup/setup-05-appstore.md) | ASC 앱 생성 → 앱 정보 → 심사 대비 3종 → 제출 |

## 핵심 원칙 (요약)

- **iOS만 배포**, 빌드는 **로컬 Xcode** (클라우드 안 씀 → 로그를 평문으로 즉시 확인).
- 스타일은 `theme/colors.ts` 단일 출처, hex 직접 금지, NativeWind 안 씀. 아이콘 Ionicons만, **이모지 금지**.
- 서버 통신은 TanStack Query 훅으로 캡슐화. Supabase 직접 호출 금지.
- 클라이언트엔 **anon key만**. service_role·외부 API 키는 **Edge Function secrets에만**.
- 모든 테이블·버킷에 **RLS 필수** (`auth.uid() = user_id`).
- 계정 생성 앱은 **앱 내 계정 삭제** + **Sign in with Apple** + **심사자 데모 계정** 필수 (App Store 리젝 대비).

## 함정 빠른 참조

- **pnpm 쓰면** `babel-preset-expo` 해석 실패로 번들이 깨진다 → npm 권장, 또는 `.npmrc`에 `node-linker=hoisted`. (자세히: [01번](./setup/setup-01-project.md))
- **reanimated 4** 는 `react-native-worklets` + `react-native-worklets/plugin` 필수.
- **`.env` 값**은 Release 번들에 빌드 시점 인라인 → 바꾸면 `--reset-cache`/Clean Build.
