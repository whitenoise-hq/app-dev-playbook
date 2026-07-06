# 02. Supabase 세팅

> 백엔드(Auth·Postgres·Storage·Edge Functions) 세팅 순서와 필수 보안 패턴.
> 원칙: 클라이언트엔 **anon key만**. service_role·OpenAI 등 비밀키는 **Edge Function secrets에만**. 모든 테이블·버킷에 **RLS 필수**.

---

## 1. 프로젝트 생성 & env 연결

1. [supabase.com](https://supabase.com) 대시보드에서 프로젝트 생성.
2. **Project Settings → API** 에서 값 확인:
   - **Project URL** → `https://<ref>.supabase.co`
   - **anon / public** key (긴 JWT `eyJ...`). ⚠️ `service_role`(secret)은 클라이언트에 **절대** 넣지 않기.
3. `.env` (gitignore됨):
   ```
   EXPO_PUBLIC_SUPABASE_URL=https://<ref>.supabase.co
   EXPO_PUBLIC_SUPABASE_ANON_KEY=eyJ...
   ```

## 2. 클라이언트 (lib/supabase.ts)

```ts
import { createClient } from '@supabase/supabase-js'
import AsyncStorage from '@react-native-async-storage/async-storage'
import 'react-native-url-polyfill/auto'

const supabaseUrl = process.env.EXPO_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey, {
  auth: {
    storage: AsyncStorage,        // RN 세션 저장
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,    // RN은 URL 세션 감지 끔
  },
})
```
> 컴포넌트에서 `supabase`를 직접 부르지 말고 `hooks/`(TanStack Query)로 래핑.

## 3. 마이그레이션 방식

- 로컬 CLI: `supabase/migrations/*.sql` 로 버전 관리 → `supabase db push`.
- 또는 대시보드 **SQL Editor**에 직접 실행.
- 어느 쪽이든 **SQL을 `supabase/migrations/`에 남겨** 재현 가능하게.

## 4. 테이블 + RLS (핵심 패턴)

테이블 만들 때마다 **RLS 켜고 소유자 정책**을 건다. 하나라도 빠지면 anon key로 뚫린다.

```sql
create table public.<table> (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references auth.users(id) on delete cascade,
  created_at timestamptz not null default now()
  -- ... 컬럼
);

create index <table>_user_id_idx on public.<table>(user_id);

-- RLS 필수
alter table public.<table> enable row level security;

create policy "select own" on public.<table> for select using (auth.uid() = user_id);
create policy "insert own" on public.<table> for insert with check (auth.uid() = user_id);
create policy "update own" on public.<table> for update using (auth.uid() = user_id);
create policy "delete own" on public.<table> for delete using (auth.uid() = user_id);

-- Data API 역할 권한 (RLS와 별개로 GRANT 필요)
grant select, insert, update, delete on public.<table> to anon, authenticated, service_role;
```

## 5. Storage (버킷 + 소유자 정책)

파일은 `{uid}/파일명` 경로로 올려서 소유자를 폴더 첫 세그먼트로 판별한다.

```sql
insert into storage.buckets (id, name, public) values ('<bucket>', '<bucket>', false);

create policy "upload own" on storage.objects for insert
  with check (bucket_id = '<bucket>' and auth.uid()::text = (storage.foldername(name))[1]);

create policy "read own" on storage.objects for select
  using (bucket_id = '<bucket>' and auth.uid()::text = (storage.foldername(name))[1]);
```
- 공유 기능처럼 **공개 읽기**가 필요하면: 버킷 `public = true` + `for select using (bucket_id = '<bucket>')`. (민감하지 않은 이미지에 한해.)
- 이미지는 **압축(WebP, ~1080–1440px)** 후 업로드. 원본 비압축 영구 저장 금지.

## 6. 계정 삭제 RPC (App Store 5.1.1(v) 필수)

계정 생성 앱은 **앱 내 계정 삭제**를 반드시 제공. `security definer`로 본인 데이터를 완전 삭제.

```sql
create or replace function public.delete_account()
returns void
language plpgsql
security definer
set search_path = public
as $$
declare uid uuid := auth.uid();
begin
  -- 스토리지 고아 파일까지 삭제
  delete from storage.objects
   where bucket_id = '<bucket>' and (storage.foldername(name))[1] = uid::text;
  delete from public.<table> where user_id = uid;
  delete from auth.users where id = uid;   -- 계정 자체 삭제
end;
$$;

grant execute on public.delete_account() to authenticated;
```
- 클라이언트: `await supabase.rpc('delete_account')` → 이후 로그아웃.
- ⚠️ 테이블·버킷이 여러 개면 여기서 **전부** 지워야 고아 데이터가 안 남는다.

## 7. Edge Functions (비밀키가 필요한 외부 API)

OpenAI 같은 키는 클라이언트에 절대 노출 금지 → Edge Function 경유.

```bash
supabase functions new <name>
supabase secrets set OPENAI_API_KEY=sk-...      # 비밀키는 secrets에만
supabase functions deploy <name>
```
- 계산·판정 로직은 `supabase/functions/_shared/`에 **단일 출처**로 두고 복붙 금지.
- AI는 원자료(예: 스탯)만 만들고, 최종 점수/등급은 **코드로 계산**(상향 편향 방지).

---

## 보안 체크리스트

- [ ] 모든 테이블 RLS 활성화 + `auth.uid() = user_id` 정책
- [ ] 클라이언트엔 `EXPO_PUBLIC_SUPABASE_URL` + anon key만
- [ ] service_role·OpenAI 키는 Edge Function secrets에만 (`.env`/커밋 금지)
- [ ] `delete_account` RPC 제공 (모든 테이블+스토리지 정리)
- [ ] 이미지 압축 후 저장, 소유자 경로(`{uid}/...`)

---

이전: [01. 프로젝트 세팅](./setup-01-project.md) · 다음: [03. 인증 세팅](./setup-03-auth.md)