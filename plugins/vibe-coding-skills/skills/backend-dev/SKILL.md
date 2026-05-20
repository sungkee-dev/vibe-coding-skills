---
name: backend-dev
description: |
  CDN React 프로젝트의 Supabase + Vercel 백엔드 연동 가이드.
  Supabase 설정, 로그인 구현, 데이터 연동, 사진 저장 작업을 할 때 반드시 이 스킬을 사용할 것.
  DB 연동, 인증, 파일 업로드, Vercel 배포 등 백엔드 관련 모든 작업에 트리거.
---

# Supabase + Vercel 백엔드 연동

## 작업 전 필수: 기존 데이터 구조 파악

작업 전에 반드시 데이터 파일(data.jsx 또는 유사 파일)을 읽어서
기존 데이터 구조를 파악하고 Supabase 스키마를 그에 맞게 설계한다.

---

## 기본 설정

**index.html에 CDN 추가** (React 앞에):
```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

**src/db.jsx 생성** (theme.jsx 바로 뒤에 로드):
```jsx
const { createClient } = supabase;
const DB = createClient('https://xxxx.supabase.co', 'eyJ...');
Object.assign(window, { DB });
```

자세한 내용은 `references/backend.md` 참고.

---

## 로그인 (사업자 단독 사용)

사용자가 1인이므로 Supabase 대시보드에서 계정을 직접 생성하고,
앱에서는 이메일/비밀번호 로그인만 구현한다.

```jsx
// 로그인
await DB.auth.signInWithPassword({ email, password });

// 앱 진입 시 로그인 여부 확인 — 미로그인이면 로그인 화면으로
DB.auth.onAuthStateChange((event, session) => {
  if (!session) showLoginScreen();
});
```

Supabase 대시보드 → Authentication → Users에서 계정 직접 생성.

---

## 보안 (RLS)

모든 테이블과 Storage 버킷에 RLS를 활성화하고,
로그인한 사용자만 접근 가능하도록 설정한다.

```sql
alter table my_table enable row level security;
create policy "auth only" on my_table
  for all using (auth.role() = 'authenticated');
```

---

## 사진 관리

피부 사진은 민감한 개인정보이므로 아래 원칙을 따른다:
- 업로드 전 압축 + WebP 변환
- 비공개 버킷 저장
- DB에는 URL 대신 파일 경로(path) 저장
- 화면 열 때마다 Signed URL 재발급

자세한 패턴은 `references/backend.md` 참고.

---

## Vercel 배포

정적 파일만 있으면 별도 설정 없이 자동 감지.
Supabase URL과 anon key는 `db.jsx`에 직접 작성
(anon key는 공개 가능하지만 RLS 설정이 반드시 선행되어야 함).
