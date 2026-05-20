# Supabase + Vercel 상세 가이드

## 1. DB 스키마 설계

작업 전에 프로젝트의 데이터 파일(data.jsx 등)을 읽어서 기존 구조를 파악한 뒤,
그에 맞는 Supabase 테이블을 설계한다.

일반적인 패턴:
```sql
create table my_table (
  id uuid primary key default gen_random_uuid(),
  -- 기존 데이터 구조의 필드들...
  created_at timestamptz default now()
);
```

**컬럼명 주의**: Supabase는 snake_case (`skin_type`), JS 객체는 camelCase (`skinType`) — 변환 필요.

---

## 2. 로그인 구현 (src/auth.jsx)

```jsx
// 로그인
const { error } = await DB.auth.signInWithPassword({ email, password });

// 로그아웃
await DB.auth.signOut();

// 현재 세션 확인
const { data: { session } } = await DB.auth.getSession();

// 앱 진입 시 로그인 체크
DB.auth.onAuthStateChange((event, session) => {
  if (!session) showLoginScreen(); // 로그인 화면으로 전환
});
```

---

## 3. 데이터 로딩 패턴

```jsx
function MyScreen({ nav, mobile }) {
  const [data, setData] = React.useState([]);
  const [loading, setLoading] = React.useState(true);

  React.useEffect(() => {
    DB.from('my_table')
      .select('*')
      .order('created_at', { ascending: false })
      .then(({ data, error }) => {
        if (!error) setData(data || []);
        setLoading(false);
      });
  }, []);

  if (loading) return (
    <div style={{ padding: '60px 24px', textAlign: 'center', color: t.inkFaint }}>
      불러오는 중...
    </div>
  );
}
```

---

## 4. 데이터 쓰기 패턴

```jsx
// 추가
await DB.from('my_table').insert({ field1, field2 });

// 수정
await DB.from('my_table').update({ field1 }).eq('id', id);

// 삭제
await DB.from('my_table').delete().eq('id', id);
```

---

## 5. 사진 업로드 — 압축 + WebP + 비공개 저장

**index.html에 압축 라이브러리 추가** (Supabase CDN 앞에):
```html
<script src="https://cdn.jsdelivr.net/npm/browser-image-compression@2/dist/browser-image-compression.js"></script>
```

**업로드 패턴**:
```jsx
async function uploadPhoto(file, folder, phase) {
  // 1. 압축 + WebP 변환 (5MB → ~300KB)
  const compressed = await imageCompression(file, {
    maxSizeMB: 0.8,
    maxWidthOrHeight: 1920,
    useWebWorker: true,
    fileType: 'image/webp',
  });

  // 2. 비공개 버킷에 업로드
  const path = `${folder}/${Date.now()}_${phase}.webp`;
  const { error } = await DB.storage.from('photos').upload(path, compressed);
  if (error) throw error;

  return path; // URL이 아닌 경로(path)를 DB에 저장
}
```

**Supabase Storage 버킷 설정**:
- Public 버킷 OFF (비공개) — 피부 사진은 민감한 개인정보
- RLS: authenticated 사용자만 접근

---

## 6. Signed URL — 화면 열 때마다 재발급

DB에는 path를 저장하고, 화면에 표시할 때 URL을 동적으로 생성한다.
Signed URL은 만료되므로 화면 열 때마다 새로 발급해야 한다.

```jsx
// 단일 URL
const { data } = await DB.storage
  .from('photos')
  .createSignedUrl(path, 60 * 60 * 24 * 7); // 7일 유효
const url = data.signedUrl;

// 여러 URL 한번에
const { data } = await DB.storage
  .from('photos')
  .createSignedUrls(paths, 60 * 60 * 24 * 7);
```

---

## 7. 사진 삭제 — Storage + DB 동시 처리

```jsx
async function deletePhoto(path, recordId, phase, index) {
  // 1. Storage에서 파일 삭제
  await DB.storage.from('photos').remove([path]);

  // 2. DB에서 해당 항목 제거
  const { data } = await DB.from('treatments')
    .select('photos').eq('id', recordId).single();
  data.photos[phase].splice(index, 1);
  await DB.from('treatments').update({ photos: data.photos }).eq('id', recordId);
}
```

---

## 8. Vercel 배포

정적 파일만 있으면 별도 설정 불필요. `vercel --prod` 한 번으로 배포.

선택사항 `vercel.json`:
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```
