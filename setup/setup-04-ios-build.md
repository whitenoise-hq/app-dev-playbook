# 04. iOS 빌드·배포 (로컬 Xcode)

> 로컬 Mac의 Xcode로 빌드해서 App Store에 올리는 순서. **EAS 클라우드 안 씀.**
> 장점: 빌드 큐 없음 / 증분 빌드로 빠름 / 빌드 로그를 평문으로 즉시 확인.

---

## 0. 사전 요구 (한 번만)

- Mac + **Xcode** (`xcodebuild -version`)
- **CocoaPods** (`pod --version`, 없으면 `brew install cocoapods`)
- **Apple Developer Program** 멤버십 ($99/년, Individual 가능)
- Xcode에 Apple ID 로그인: **Settings(⌘,) → Accounts**
- App Store Connect에 앱 레코드 생성 — [05. App Store 제출](./setup-05-appstore.md) 참조

---

## 1. env 확인

Release 번들은 `EXPO_PUBLIC_*`를 **빌드 시점에 코드로 인라인**한다. `.env`가 없으면 서버 연결 없이 출시된다.

```bash
# .env 존재 확인 (없으면 만든다 — 02번 문서 참조)
cat .env
```

## 2. 네이티브 프로젝트 생성 (prebuild)

`ios/` 폴더가 없으면(managed) 생성:

```bash
npx expo prebuild --platform ios --clean
```
- `app.json`을 읽어 `ios/` 생성 + `pod install` 자동.
- `ios/`는 gitignore(`/ios`). 네이티브 설정 바꿀 때마다 `--clean`으로 재생성(CNG 방식).

## 3. Xcode 열기 & 서명

```bash
open ios/*.xcworkspace     # .xcodeproj 아님!
```
1. 좌측 최상단 프로젝트 → TARGETS → **Signing & Capabilities**
2. **Automatically manage signing** 체크
3. **Team** 선택
4. Signing Certificate / Provisioning Profile이 **에러 없이** 채워지면 OK

> **"no devices" 서명 에러가 나면** 아이폰을 케이블로 연결(→"신뢰")하면 Xcode가 기기를 등록해 해결된다. App Store 배포 프로파일 자체는 기기가 불필요하지만 자동 서명이 개발 프로파일도 만들려다 막히는 것.

## 4. 아카이브

1. 상단 기기 선택을 **Any iOS Device (arm64)** 로 (시뮬레이터면 Archive 비활성)
2. 메뉴 **Product → Archive** (좌측 "Start Build"는 Xcode Cloud이므로 쓰지 말 것)
3. 첫 빌드는 네이티브 전체 컴파일 + JS 번들링 → 몇 분
4. 성공 시 **Organizer** 창 자동 오픈

## 5. 업로드

Organizer에서:
1. 아카이브 선택 → **Distribute App**
2. **App Store Connect** → **Upload**
3. 기본값 그대로 **Next** → 서명 자동 처리 → **Upload**
4. "Uploaded to Apple" → 성공

## 6. 이후

- App Store Connect에서 몇 분~1시간 "처리 중" 후 **TestFlight**에 빌드 등장.
- **먼저 TestFlight로 실기기 테스트** (서버 연결/로그인/OAuth가 실전에서 되는지).
- 심사 제출: [05. App Store 제출](./setup-05-appstore.md)

---

## 버전 / 빌드 번호

- `app.json`의 `version`(예 1.0.0)과 `ios.buildNumber`.
- 같은 버전 재업로드 시 **buildNumber만 올림**(중복 불가). Xcode Archive가 `app.json` 값을 그대로 사용.

## 재빌드 (2회차부터)

- 코드만 바꿈 → `ios/` 유지, Xcode에서 바로 **Product → Archive** (증분, 빠름).
- 네이티브 설정(plugins/아이콘/permission) 바꿈 → `npx expo prebuild -p ios --clean` 다시.
- `.env` 값 바꿈 → **Clean Build Folder(⇧⌘K)** 후 Archive (Metro 캐시 무효화).

## 빌드 실패 시 (평문 로그)

로컬 빌드의 장점: 로그가 평문. 서명 없이 번들/컴파일만 검증:

```bash
xcodebuild -workspace ios/*.xcworkspace -scheme <scheme> -configuration Release \
  -sdk iphoneos -destination 'generic/platform=iOS' \
  CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO build 2>&1 | tee /tmp/xc.log
grep -iE "error:|BUILD FAILED|Cannot find module" /tmp/xc.log
```

---

이전: [03. 인증 세팅](./setup-03-auth.md) · 다음: [05. App Store 제출](./setup-05-appstore.md)