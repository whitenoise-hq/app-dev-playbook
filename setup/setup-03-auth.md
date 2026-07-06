# 03. 인증 세팅 (카카오 / 애플)

> Supabase Auth로 소셜 로그인. **카카오는 OAuth 웹 플로우**, **애플은 네이티브 플로우**로 방식이 다르다.
> App Store 정책: 소셜 로그인을 제공하면 **Sign in with Apple도 함께 제공**해야 함(4.8). provider를 추상화해 쉽게 추가할 수 있게 둔다.

---

## 0. provider 추상화 (lib/auth.ts)

새 provider 추가 시 이 파일만 손대도록 한 곳에 모은다.

```ts
import { makeRedirectUri } from 'expo-auth-session'
import * as WebBrowser from 'expo-web-browser'
import * as AppleAuthentication from 'expo-apple-authentication'
import { supabase } from './supabase'
import type { Provider } from '@supabase/supabase-js'

const redirectTo = makeRedirectUri()   // 앱 scheme으로 복귀 (app.json "scheme" 필요)

// 웹 OAuth 플로우 (카카오 등)
export async function signInWithProvider(provider: Provider) {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider,
    options: { redirectTo, skipBrowserRedirect: true },
  })
  if (error) throw error
  if (!data.url) throw new Error('OAuth URL을 받지 못했습니다')

  const result = await WebBrowser.openAuthSessionAsync(data.url, redirectTo)
  if (result.type === 'success' && result.url) {
    const params = new URLSearchParams(new URL(result.url).hash.substring(1))
    const access_token = params.get('access_token')
    const refresh_token = params.get('refresh_token')
    if (access_token && refresh_token) {
      const { error } = await supabase.auth.setSession({ access_token, refresh_token })
      if (error) throw error
    }
  }
}

export const signInWithKakao = () => signInWithProvider('kakao')
export const signOut = async () => { const { error } = await supabase.auth.signOut(); if (error) throw error }
export const deleteAccount = async () => { const { error } = await supabase.rpc('delete_account'); if (error) throw error }
```

## 1. app.json — scheme + 애플 capability

```jsonc
{
  "expo": {
    "scheme": "<app>",                 // redirect 복귀용. 필수.
    "ios": { "usesAppleSignIn": true } // 애플 로그인 capability (prebuild가 네이티브에 반영)
  }
}
```

---

## 2. 카카오 로그인 (웹 OAuth)

**Supabase는 kakao provider를 기본 지원.**

1. [카카오 개발자](https://developers.kakao.com) → 애플리케이션 생성 → **REST API 키** 확보.
2. 카카오 앱 설정 → **카카오 로그인 활성화** → **Redirect URI** 등록:
   ```
   https://<ref>.supabase.co/auth/v1/callback
   ```
3. 동의항목(닉네임/이메일 등) 설정.
4. Supabase 대시보드 → **Authentication → Providers → Kakao** → REST API 키 + Client Secret 입력, 활성화.
5. 클라이언트: `signInWithKakao()` 호출.

> 카카오는 웹 시트(`WebBrowser`)로 진행 → 성공 시 URL 해시의 토큰으로 세션 설정.

---

## 3. 애플 로그인 (네이티브 플로우)

카카오와 달리 **네이티브 시트 → identityToken → Supabase `signInWithIdToken`**. 웹 리다이렉트 아님. **실기기/dev build에서만 동작(Expo Go 불가).**

```ts
export async function signInWithApple() {
  const credential = await AppleAuthentication.signInAsync({
    requestedScopes: [
      AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
      AppleAuthentication.AppleAuthenticationScope.EMAIL,
    ],
  })
  if (!credential.identityToken) throw new Error('Apple identityToken을 받지 못했습니다')
  const { error } = await supabase.auth.signInWithIdToken({
    provider: 'apple',
    token: credential.identityToken,
  })
  if (error) throw error
}
```

**Apple Developer / Supabase 설정:**
1. Apple Developer → **Identifiers**에서 App ID에 **Sign In with Apple** capability 추가.
2. **Service ID** 생성 + **Key**(Sign in with Apple) 발급(.p8).
3. Supabase → **Authentication → Providers → Apple** → Service ID / Team ID / Key ID / .p8 입력, 활성화.
4. 클라이언트 UI: iOS에서만 애플 버튼 노출.
   ```ts
   const [appleAvailable, setAppleAvailable] = useState(false)
   useEffect(() => {
     if (Platform.OS === 'ios') AppleAuthentication.isAvailableAsync().then(setAppleAvailable)
   }, [])
   ```
5. 취소 처리: `error.code === 'ERR_REQUEST_CANCELED'` 는 오류로 보지 않음.

---

## 4. 세션 & 온보딩

- 세션 구독 훅(예: `useSession`)으로 로그인 상태 감지 → 보호 라우트 자동 리다이렉트.
- 닉네임 등 추가 정보는 `supabase.auth.updateUser({ data: { nickname } })` 로 user_metadata에 저장 → `USER_UPDATED` 이벤트로 구독 갱신.

## 5. 계정 삭제 (설정 화면)

```ts
await deleteAccount()   // rpc('delete_account')
await signOut()
```
- 앱 내에서 접근 가능한 위치(설정)에 반드시 배치 (App Store 5.1.1(v)). 상세: [02. Supabase 세팅](./setup-02-supabase.md#6-계정-삭제-rpc)

---

이전: [02. Supabase 세팅](./setup-02-supabase.md) · 다음: [04. iOS 빌드·배포](./setup-04-ios-build.md)