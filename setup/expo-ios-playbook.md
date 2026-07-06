# Expo + iOS 앱 개발 플레이북

> 새 Expo(React Native) 앱을 만들 때 참고하는 개인 레퍼런스. 포포노트 개발에서 검증한 세팅·명령어·함정 모음.
> **방침: 앞으로 iOS(App Store)만 배포. Android는 빌드/제출하지 않는다.**
> 새 프로젝트 시작 시 이 파일을 복사해서 프로젝트에 맞게 값만 바꾼다.

---

## 1. 기술 스택 (기본 선택)

| 영역 | 선택 |
| --- | --- |
| 프레임워크 | React Native + Expo, TypeScript |
| 라우팅 | Expo Router (파일 기반) |
| 스타일 | RN `style` prop + `StyleSheet.create`. 색상은 `theme/colors.ts` 단일 출처(hex 직접 박지 않기). NativeWind 안 씀 |
| 서버 상태 | TanStack Query (React Query) |
| 백엔드 | Supabase (Auth / Postgres / Storage / Edge Functions) |
| 인증 | 카카오/애플 OAuth (Supabase Auth) |
| AI | OpenAI 등 — **키는 Edge Function secrets에만**, 클라이언트 금지 |
| 아이콘 | Ionicons(`@expo/vector-icons`). **이모지 금지**(기기별 렌더링 불일치) |

---

## 2. 프로젝트 생성 & 초기 세팅

```bash
# 생성
npx create-expo-app@latest my-app --template
cd my-app

# 핵심 패키지
npx expo install @supabase/supabase-js @react-native-async-storage/async-storage react-native-url-polyfill
pnpm add @tanstack/react-query
npx expo install expo-router expo-image-picker expo-media-library expo-web-browser expo-splash-screen
npx expo install @react-native-community/datetimepicker react-native-safe-area-context
```

- `app.json`에 `scheme`, `ios.bundleIdentifier`, plugins(expo-router, expo-splash-screen 등) 설정.
- `experiments: { typedRoutes: true }` 권장(라우트 타입 안전).
- 아이콘/스플래시: `assets/images/icon.png`(1024+ 정사각), splash 이미지.

### 폴더 구조
```
app/         # Expo Router 화면 ((tabs) 그룹 포함)
components/   # 재사용 UI
components/ui/ # Text·Card·Button·Screen 등 토큰 적용 공용 컴포넌트
lib/          # supabase 클라이언트, query-client, 유틸
hooks/        # React Query 커스텀 훅
theme/        # colors.ts (색상 단일 출처)
types/        # 공용 타입
supabase/     # migrations, Edge Functions
docs/         # 기획/배포 문서
```

---

## 3. 환경변수 / 보안

- 클라이언트엔 **Supabase URL + anon key만**. `.env`에 `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY`.
- `.env`는 **커밋 금지**(.gitignore). `.env.example`엔 플레이스홀더만.
- **API 키·시크릿(OpenAI 등)은 Edge Function 환경변수(secrets)에만.** 클라이언트 절대 금지.
- 계정·자격증명 식별자(Apple ID, Team ID, 인증서 Serial 등)는 `docs/*.local.md`로 분리하고 `*.local.md`를 .gitignore.
- 모든 테이블에 **RLS 적용**. 사용자는 자기 그룹 데이터만, 본인 레코드만 수정·삭제.

```bash
# Edge Function secret 설정
npx supabase secrets set OPENAI_API_KEY=sk-...
```

---

## 4. EAS 설정

전역 설치: `npm i -g eas-cli` / 확인 `npx eas whoami`

`eas.json` (iOS 중심):
```json
{
  "cli": { "version": ">= 3.13.2", "appVersionSource": "local" },
  "build": {
    "development": { "developmentClient": true, "distribution": "internal" },
    "preview": { "distribution": "internal" },
    "production": { "autoIncrement": true }
  },
  "submit": {
    "production": {
      "ios": { "ascAppId": "<App Store Connect 앱 숫자 ID>" }
    }
  }
}
```
- `appVersionSource: "local"` → 버전 출처는 `app.json`. autoIncrement가 빌드 후 app.json에 써넣으므로 그 변경을 커밋.
- `ascAppId`만 eas.json에 둠(비밀 아님). **Apple ID/Team ID는 넣지 말 것**(커밋되는 파일).

---

## 5. iOS 배포 절차

### 한 번만 (재사용됨)
- 최초 `eas build -p ios --profile production` 시 EAS가 배포 인증서·프로비저닝 프로파일 자동 발급·보관(만료 전까지 재사용).
- 최초 `eas submit -p ios` 시 App Store Connect API Key(역할 APP_MANAGER) 자동 생성·저장.
- `ascAppId`(App Store Connect → 앱 → 앱 정보의 10자리 Apple ID)를 eas.json에 적어두면 이후 `--non-interactive` 제출 가능. 없으면 첫 submit이 대화형 Apple 로그인을 요구함.

### 매 배포
```bash
# 1. 버전 올리기: app.json의 expo.version 손으로 (1.0.0 → 1.0.1). buildNumber는 autoIncrement가 자동.
# 2. 빌드
npx eas build --platform ios --profile production
# 3. 제출 (ascAppId 설정돼 있으면 프롬프트 없이)
npx eas submit --platform ios --profile production --latest --non-interactive
# 빌드+제출 한 번에:
npx eas build --platform ios --profile production --auto-submit
```
- 빌드 ~10~20분, 제출 후 Apple 처리 5~30분 → TestFlight 표시.
- 빌드 후 `git status`에 app.json(buildNumber) 변경 뜨면 커밋.
- 현황: https://expo.dev/accounts/<account>/projects/<slug> · 빌드 목록 `npx eas build:list --platform ios`

### 버전 관리 원칙 (중요)
- `expo.version`(마케팅 버전): iOS 공통, 기능/수정 릴리스 시 **손으로** 올림.
- `expo.ios.buildNumber`: **autoIncrement가 자동 +1**, 손 안 댐. 각 스토어 안에서 이전보다 크기만 하면 됨.
- (Android 안 하므로 versionCode 신경 안 씀)

---

## 6. App Store Connect 체크리스트

- `app.json` → `ios.infoPlist`:
  - `ITSAppUsesNonExemptEncryption: false` (표준 암호화만 쓸 때 — 수출규정 질문 스킵)
  - 사용 권한 문구: `NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription`, 카메라 쓰면 `NSCameraUsageDescription` 등 (없으면 심사 리젝)
- **연령 등급 설문(제공 기능)**: 과한 등급(17+) 피하려면 정확히 답할 것
  - **제한 없는 웹 접근**: 고정 링크만 열면(앱 내 범용 브라우저 아님) → **아니요** ← 이게 17+의 흔한 원인
  - 사용자 생성 콘텐츠: 실제 UGC면 **예**(가이드라인 1.2 — 신고/차단/문의 기능 요구). 비공개 그룹 한정이면 위험 낮음
  - 메시지/채팅: 게시물 댓글은 채팅 아님 → 보통 아니요
  - 광고 없으면 아니요
- 스토어 등록: 스크린샷(필수, 실제 출시 시), 앱 아이콘은 **빌드 바이너리에서 자동**(별도 업로드 불필요).
- 개인정보처리방침 URL 필요(앱이 개인정보 수집 시).

---

## 7. Supabase 패턴 (검증됨)

### RLS 닭-달걀 (초대코드 등)
- "가입하려면 읽어야 하는데 멤버여야 읽을 수 있는" 경우 → **security definer 함수**로 RLS 우회, 올바른 입력에만 최소 정보 반환.
```sql
create or replace function public.verify_invite_code(_code text)
returns table (family_id uuid, pet_name text)
language sql stable security definer as $$ ... $$;
grant execute on function public.verify_invite_code(text) to authenticated;
```

### 계정 삭제도 security definer
- owner 그룹 cascade 삭제 + 멤버 탈퇴를 RPC(`delete_account`)로.

### 마이그레이션 적용
```bash
npx supabase db push --dry-run   # 무엇이 적용될지 먼저 확인
npx supabase db push
npx supabase migration list      # Local/Remote 일치 확인
```

### Realtime (멀티유저 실시간 동기화)
1. 마이그레이션: 대상 테이블을 publication에 추가 + DELETE 정밀 처리 필요하면 replica identity full
   ```sql
   alter publication supabase_realtime add table public.<t>;  -- do/except로 멱등 가드
   alter table public.comments replica identity full;
   ```
2. 클라이언트: createClient에 `realtime: { params: { eventsPerSecond: 10 } }`.
3. **RLS 적용 필수**: 로그인/토큰갱신 시 `supabase.realtime.setAuth(session.access_token)` (안 하면 anon이라 RLS 통과 0건 수신).
4. 가족/그룹 단일 채널 구독 훅 → 이벤트 수신 시 관련 queryKey `invalidateQueries`. 그룹 id 컬럼 있는 테이블은 서버 필터 `col=eq.<id>`, 없는 테이블은 RLS에 의존(**cross-group 누수 2계정 테스트 필수**).

---

## 8. React Query 패턴 (검증됨)

`lib/query-client.ts`:
```js
export const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 1000 * 60 * 5, retry: 1 } },
})
// RN엔 window focus가 없으니 AppState를 focusManager에 연결(백그라운드 복귀 시 refetch)
AppState.addEventListener('change', (s) => focusManager.setFocused(s === 'active'))
```

데이터 신선도 3중 전략:
1. mutation `onSuccess`/`onSettled`에서 `invalidateQueries`.
2. Realtime로 타 기기 변경 반영(7번).
3. 화면 포커스/앱 복귀 refetch — `useFocusEffect`(expo-router) 기반 `useRefetchOnFocus(keys)` 훅 + AppState focusManager.

함정·교훈:
- **쿼리 키에 스코프 id 포함**: `['diary', familyId]`처럼. 사용자 전환 시 캐시 충돌 방지.
- **로그아웃 시 `queryClient.clear()`**: 이전 사용자 데이터 잔존/노출 방지(같은 계정 재로그인 시 stale 노출).
- **온보딩 직후 `invalidateQueries()`**: 가입 전 화면이 잠깐 마운트되며 빈 값이 캐시될 수 있음 → 가입 완료 후 무효화.
- **날짜를 모듈 상수로 계산하지 말 것**(`const TODAY = todayStr()`): 자정 넘겨 앱 재개 시 어제로 고정됨. AppState로 갱신되는 `useToday()` 훅 사용.
- 탭(`<Tabs>`)은 전환해도 unmount 안 됨 → 재진입 refetch는 `useFocusEffect` 필요.

---

## 9. React Native UI 흔한 함정 (iOS)

- **커스텀 폰트 굵기**: 커스텀 `fontFamily`에 `fontWeight`를 같이 주면 **Android·일부 환경에서 시스템 폰트로 폴백**(굵기별 파일이 별도 패밀리로 등록되므로). 굵기는 **굵기별 fontFamily**로 지정하고 fontWeight 제거. 공용 `Text` 컴포넌트에서 fontWeight→fontFamily 매핑 처리해두면 전역 안전.
- **DateTimePicker 다크모드**: iOS 스피너는 글자색이 시스템 테마를 따라, 흰 모달 배경에선 다크모드 시 글자가 안 보임 → `themeVariant="light"` + `textColor` 명시.
- **키보드 회피**:
  - 하단 고정 입력 + 스크롤 화면: **화면 전체를 `KeyboardAvoidingView`(iOS `behavior="padding"`)로 감싼다**. 입력 바만 감싸면 불안정.
  - 스크롤뷰 맨 아래 입력: ScrollView `automaticallyAdjustKeyboardInsets`(iOS) + 열 때 `scrollToEnd`.
  - 입력 바 하단 여백: 키보드 올라오면 SafeArea bottom inset 제거(상하 균형), 닫히면 유지.
  - 전송/엔터 시 `Keyboard.dismiss()`.
- **이미지 비율**: 정사각 저장 사진은 `width:'100%' + aspectRatio:1`로 표시(고정 height는 잘림).
- **시뮬레이터 키보드 안 뜸**: I/O → Keyboard → Connect Hardware Keyboard 해제(⌘⇧K).
- 색상은 `theme/colors`에서만, hex 직접 사용 금지. 이모지 대신 Ionicons.

---

## 10. 자주 쓰는 명령어

```bash
# 개발
npx expo start            # 개발 서버 (dev client/Expo Go)
npx expo start -c         # 캐시 클리어
npx tsc --noEmit          # 타입 체크

# EAS (iOS)
npx eas build --platform ios --profile production
npx eas submit --platform ios --profile production --latest --non-interactive
npx eas build:list --platform ios --limit 5
npx eas credentials       # 인증서/키 확인

# Supabase
npx supabase db push --dry-run
npx supabase db push
npx supabase migration list
npx supabase secrets set KEY=value
```

---

## 11. 릴리스 전 체크리스트

- [ ] `app.json`: version 올림, bundleId, 권한 문구, `ITSAppUsesNonExemptEncryption`
- [ ] 모든 테이블 RLS + 본인 레코드만 수정/삭제 정책
- [ ] 클라이언트에 시크릿 없음(anon key만), `.env` 미커밋
- [ ] 연령 등급 설문 정확히(제한없는 웹접근=아니요 등)
- [ ] 개인정보처리방침 URL
- [ ] 다크모드/키보드/폰트 함정 확인(9번)
- [ ] 멀티유저면 realtime cross-group 누수 2계정 테스트

---

## 12. Sign in with Apple 설정

App Store 가이드라인 4.8: 소셜 로그인(카카오)을 제공하면 Apple 로그인도 동등하게 제공해야 리젝을 피한다.
구현은 **네이티브 플로우**: `expo-apple-authentication`의 `signInAsync` → `identityToken`을 `supabase.auth.signInWithIdToken({ provider: 'apple', token })`로 교환. (카카오의 OAuth 웹 플로우와 다름)

### 코드/설정 (완료)
- `expo-apple-authentication` 설치, `app.json` `plugins`에 추가 + `ios.usesAppleSignIn: true`.
- `lib/auth.ts`의 `signInWithApple()`, `app/(auth)/login.tsx`의 Apple 버튼(`isAvailableAsync()`로 노출 게이팅).

### 빌드 (필수)
플러그인 추가 후 **네이티브 재생성·리빌드 전에는 버튼이 안 보인다**(`isAvailableAsync()`가 false). Expo Go에서는 동작 불가.
```
npx expo prebuild            # 플러그인 → 엔타이틀먼트(com.apple.developer.applesignin) + bundleId 반영
npx pod-install              # 또는 prebuild가 자동
npx expo run:ios             # 실기기/dev build (시뮬레이터는 iCloud 로그인 시 동작)
```

### Apple Developer (developer.apple.com)
- App ID(`com.devwoodie.popostat`)에 **Sign In with Apple** capability 활성화.
- 웹/안드로이드까지 확장할 때만: **Services ID** 생성 + Return URL `https://djvdxacfuaztndcvxqph.supabase.co/auth/v1/callback` 등록 + Sign in with Apple **키(.p8)** 발급(Key ID/Team ID 기록).

### Supabase Dashboard (Authentication → Providers → Apple)
- **Client IDs**에 iOS **Bundle ID** `com.devwoodie.popostat` 입력 — 네이티브 `identityToken`의 `aud` 검증용. 네이티브 전용이면 이것만으로 동작.
- 웹 OAuth까지 쓸 때만 Secret Key 섹션에 Services ID/Team ID/Key ID/.p8 입력.

### 참고: 카카오 iOS
카카오는 OAuth 웹 플로우라 앱에 client id를 넣지 않는다. 카카오 Developers 콘솔에서 ① Redirect URI `https://djvdxacfuaztndcvxqph.supabase.co/auth/v1/callback` 등록, ② 플랫폼 → iOS에 Bundle ID `com.devwoodie.popostat` 등록. Supabase Kakao provider에는 REST API 키(Client ID)·시크릿 입력.
- [ ] `npx tsc --noEmit` 통과