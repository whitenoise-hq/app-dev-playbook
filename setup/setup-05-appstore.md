# 05. App Store 제출

> App Store Connect 앱 생성부터 심사 제출까지. **계정 생성/로그인 앱**은 리젝 포인트가 정해져 있으니 미리 대비한다.

---

## 1. App Store Connect 앱 생성

1. [App Store Connect](https://appstoreconnect.apple.com) → **앱 → +** → 새 앱.
2. 입력:
   - **플랫폼**: iOS
   - **Bundle ID**: `app.json`의 `ios.bundleIdentifier`와 **정확히 일치** (연결은 이 값으로만 됨)
   - 이름 / 기본 언어 / SKU
3. Bundle ID가 목록에 없으면 [developer.apple.com](https://developer.apple.com/account) → Identifiers에서 먼저 등록. (로컬 빌드/EAS가 자동 등록하기도 함.)

> 빌드는 [04. iOS 빌드·배포](./setup-04-ios-build.md)로 업로드하면 Bundle ID로 이 앱에 자동으로 붙는다.

## 2. 앱 정보 (제출 전 채우기)

- **스크린샷** (필수): 6.9"(또는 6.7") + 6.5" 등 요구 사이즈. 실기기/시뮬레이터 캡처.
- **설명 / 프로모션 텍스트 / 키워드 / 지원 URL**
- **개인정보 처리방침 URL** (계정/데이터 수집 앱은 필수)
- **App 개인정보(Privacy) 설문**: 수집하는 데이터 유형 정직하게 (예: 이메일, 사용자 콘텐츠).
- **연령 등급** 설문
- **카테고리**

## 3. Export Compliance (암호화)

- `app.json`에 `ITSAppUsesNonExemptEncryption: false`를 넣어두면 매 빌드 암호화 질문을 건너뛴다. (표준 HTTPS만 쓰는 일반 앱 기준.)

## 4. 심사 대비 (계정/로그인 앱 필수 3종)

리젝 최다 포인트. 제출 전 반드시 확인:

- [ ] **Sign in with Apple 제공** (4.8) — 다른 소셜 로그인(카카오 등)을 쓰면 애플 로그인도 함께 있어야 함. → [03. 인증](./setup-03-auth.md)
- [ ] **앱 내 계정 삭제** (5.1.1(v)) — 설정 등 앱 내에서 접근 가능한 위치에 계정 삭제 기능. `delete_account` RPC. → [02. Supabase](./setup-02-supabase.md#6-계정-삭제-rpc)
- [ ] **심사자용 데모 계정** — 로그인 벽이 있으면 **App Review Information → 로그인 필요 체크 → 데모 계정/비번 또는 로그인 방법**을 반드시 기재. 카카오/애플 로그인이 심사 환경에서 안 되면 대체 로그인이나 데모 토큰 안내를 Notes에 적는다. (안 적으면 거의 확실히 리젝.)

## 5. 제출

1. TestFlight에서 빌드 **처리 완료** 확인 + 실기기 동작 확인.
2. 앱의 **버전(예 1.0.0)** 페이지 → **빌드 추가**로 업로드한 빌드 선택.
3. 위 2~4 항목 모두 채움.
4. **심사 제출(Submit for Review)**.
5. 상태: 심사 대기 → 심사 중 → 승인/거부. 보통 1~3일.

## 6. 리젝 대응

- 거부되면 **Resolution Center**에 사유가 온다. 코드 수정 → buildNumber 올려 재업로드 → 같은 버전에 새 빌드 연결 → 재제출.
- 메타데이터 문제(스크린샷/설명 등)면 빌드 재업로드 없이 텍스트만 고쳐 재제출 가능.

---

## 제출 전 최종 체크리스트

- [ ] Bundle ID 일치 (app.json ↔ ASC)
- [ ] `.env` 실제 값으로 빌드됨 (TestFlight에서 서버 연결 확인)
- [ ] Sign in with Apple / 계정 삭제 / 데모 계정 (3종)
- [ ] 스크린샷 / 개인정보 URL / Privacy 설문
- [ ] version·buildNumber 정리

---

이전: [04. iOS 빌드·배포](./setup-04-ios-build.md) · 처음: [01. 프로젝트 세팅](./setup-01-project.md)