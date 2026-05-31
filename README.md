# Case Studio — 배포/실행 안내

## 1) GitHub Pages에 올리기
1. 이 폴더의 **내용물 전체**(product.html, index.html, dielines/, mockups/ 등)를 레포에 그대로 업로드.
   - ⚠️ product.html만 올리면 안 됨 — dielines/ · mockups/ 폴더가 같은 위치에 있어야 함.
2. 레포 Settings → Pages → Branch: main / root 로 활성화.
3. 접속: `https://<사용자명>.github.io/<레포명>/product.html`
   - 예) `https://smh1023-dev.github.io/case-studio/product.html`

## 2) 로컬에서 테스트(서버 필요)
- 이 폴더에서 터미널: `python -m http.server 8000`
- 브라우저: `http://localhost:8000/product.html`
- ❌ 파일 더블클릭(file://)은 브라우저가 로컬 이미지를 막아 칼선·목업이 안 뜸.

## 3) 처음 접속 후 할 일
- 메인 **설정**에서 OpenAI 키 저장
- product.html 하단 **이미지 엔진 설정**에서 fal.ai 키 저장
- (키는 코드가 아니라 브라우저 localStorage에 도메인별 저장 → 도메인이 바뀌면 다시 입력)

## 4) 주의
- GitHub Pages(무료)는 공개 주소. 키를 코드에 하드코딩하지 말 것(현재도 안 박혀 있음).
- 공용 PC에선 키 저장 금지.

## 페이지 구성
- index.html(메인) · product.html(제작) · learning.html · mydesign.html · evaluation.html · reels.html
- 마케팅: event.html · logo.html · admin.html (키 없음)
- dielines/ : 아이폰·갤럭시 칼선 PNG 56종
- mockups/ : Front/Side 목업 오버레이(mk-front-overlay.png, mk-side-overlay.png) + 합성용 PNG
