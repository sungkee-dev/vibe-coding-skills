---
name: frontend-dev
description: |
  CDN React + Babel Standalone 방식의 프론트엔드 개발 가이드.
  npm이나 빌드 도구 없이 정적 파일로 동작하는 React 프로젝트에서
  화면 추가, 컴포넌트 수정, UI 작업을 할 때 반드시 이 스킬을 사용할 것.
  화면 추가, UI 수정, 버그 수정, 스타일 변경, 컴포넌트 추가 등
  모든 프론트엔드 작업에 트리거.
---

# CDN React 프론트엔드 개발

## 절대 원칙

이 프로젝트는 **CDN React + Babel Standalone** 방식으로만 개발한다.
- npm install, Vite, Create React App, Next.js 등 빌드 도구 절대 사용 금지
- 새 라이브러리가 필요하면 CDN 주소로 index.html에 추가
- 모든 코드는 `.jsx` 파일에 작성하고 `index.html`에서 `<script type="text/babel">`로 로드
- 로컬 실행: `python3 -m http.server 3000`
- 배포: Vercel (정적 파일 자동 감지)

---

## 작업 전 필수: 기존 코드 파악

작업 전에 반드시 아래 파일들을 읽어서 이 프로젝트의 패턴을 파악한다.
기존 패턴과 다른 방식으로 짜면 전체 일관성이 깨진다.

- **index.html** → 파일 로드 순서, CDN 목록 확인
- **컴포넌트 파일** (ui.jsx 또는 유사 파일) → 어떤 컴포넌트가 있는지
- **테마/스타일 파일** → 색상 토큰, 테마 훅 이름
- **기존 화면 파일 하나** → 컴포넌트 구조, 스타일 패턴 확인
- **앱 셸 파일** (app.jsx 또는 유사 파일) → 내비게이션 구조 확인

---

## 공통 코딩 규칙

**1. 컴포넌트 전역 등록** — 파일 간 공유를 위해 파일 맨 끝에 필수:
```jsx
Object.assign(window, { MyComponent });
```

**2. 색상은 테마 훅 사용 — 하드코딩 금지**:
```jsx
const t = useTheme(); // 프로젝트의 테마 훅 이름 확인 후 사용
// 올바름: t.ink, t.accent, t.surface 등 기존 토큰
// 잘못됨: '#333333', 'rgba(0,0,0,0.5)' 직접 입력
```

**3. 스타일은 인라인** — CSS 파일, CSS 클래스 사용 금지:
```jsx
<div style={{ color: t.ink, padding: '16px', borderRadius: 8 }}>
```

**4. 반응형은 mobile prop** — 프로젝트에 이 패턴이 있으면 따름:
```jsx
function MyScreen({ nav, mobile }) {
  if (mobile) return <div>모바일 레이아웃</div>;
  return <div>데스크톱 레이아웃</div>;
}
```

---

## index.html 파일 로드 순서

새 파일 추가 시 의존성 순서를 반드시 지킨다.

1. index.html에서 기존 로드 순서 확인
2. 내가 의존하는 파일 뒤, 메인 앱 파일 앞에 삽입:

```html
<script type="text/babel" src="src/my-new-file.jsx"></script>
```

---

## 새 화면 추가하는 법

1. 기존 화면 파일을 참고해서 같은 구조로 `src/screen-xxx.jsx` 생성
2. `index.html`에 script 태그 추가 (메인 앱 파일 앞)
3. 앱 셸 파일(app.jsx 등)에 라우팅 분기 추가
4. 필요시 내비게이션 메뉴에 항목 추가

## 새 컴포넌트 추가하는 법

1. 기존 컴포넌트 파일(ui.jsx 등)에 추가하거나 새 파일 생성
2. 기존 컴포넌트 스타일과 동일한 패턴으로 작성
3. `Object.assign(window, { NewComponent })`로 전역 등록
4. 새 파일이면 index.html 로드 순서 확인
