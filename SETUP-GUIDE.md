# SI Story — 이벤트/로고 시스템 셋업 가이드

기존 SI Story 사이트는 그대로 두고, 아래 3개 페이지가 추가됩니다.
- **event.html** — 릴스 유입 이벤트 랜딩 (공개)
- **logo.html** — 고객 입력 → 구글폼 신청 (공개, 키 없음)
- **admin.html** — 관리자 전용: 신청자 조회 + AI 로고 생성 (본인 PC에서만)

고객 흐름: 릴스 → event.html → 카카오채널 가입 → 구글폼 작성 → 구글시트 자동저장 → (Apps Script가 AI 호출) → 결과 이메일/카카오 발송 → 케이스 구매.

---

## 1. 페이지에서 바꿔야 할 주소 (필수)

**event.html** 하단 `<script>` CONFIG:
- `KAKAO_URL` → 카카오채널 주소 (예: http://pf.kakao.com/_XXXX)
- `FORM_URL` → 구글폼 주소 (forms.gle/…)

**logo.html** 하단 CONFIG:
- `KAKAO_URL`, `SHOP_URL`(스마트스토어 등)
- `FORM_BASE` + `ENTRY.*` → 구글폼 "사전 입력된 링크"에서 entry 번호 복사

**admin.html**:
- 화면에서 "구글시트 CSV 게시 URL" 입력 (아래 4번 참고)
- OpenAI 키는 메인(index.html) 설정에서 저장한 값을 공유

---

## 2. 구글폼 필드 (순서대로 만들기)

1. 이메일 (단답)
2. 카카오채널 가입 여부 (객관식: 가입함 / 안 함)
3. 카카오채널 닉네임 (단답)
4. 불리고 싶은 애칭 (단답)
5. 좋아하는 색 (단답)
6. 가치관 3개 (단답 — 쉼표로 구분)
7. 좋아하는 동물/상징 (단답)
8. 가장 자랑스러운 순간 (장문)
9. 극복한 어려움 (장문)
10. 사은품 선택 (객관식: 그립톡 / 카드지갑)
11. 사은품 수령 여부 (객관식: 예 / 아니오)
12. 이름 (단답)
13. 연락처 (단답)
14. 주소 (장문)
15. 개인정보 수집 및 이용 동의 (체크박스 · 필수)
16. 마케팅 수신 동의 (체크박스 · 선택)

> 폼 만든 뒤 "응답 → 시트로 연결"을 누르면 구글시트가 자동 생성됩니다.

---

## 3. 구글시트 컬럼 (자동 생성 + 수동 추가)

폼 연결 시 1~16번이 자동으로 들어갑니다. 그 오른쪽에 아래 컬럼을 **수동으로 추가**하세요:

```
timestamp, email, kakao_joined, kakao_nickname, nickname, favorite_color,
value1, value2, value3, symbol, proud_moment, challenge,
gift_type, gift_required, name, phone, address,
privacy_consent, marketing_consent,
logo_status, generated_identity, generated_slogan, image_prompt, case_purchase_status
```

`logo_status` 값: 접수 / 생성중 / 완료 / 발송완료 / 케이스구매

---

## 4. admin.html에서 시트 불러오기

1. 구글시트 → 파일 → 공유 → **웹에 게시**
2. "전체 문서" 대신 응답 시트 선택, 형식 **CSV**
3. 생성된 링크를 admin.html "CSV 게시 URL"에 붙여넣기 → 저장 → 불러오기

> ⚠️ 이 링크엔 개인정보가 들어갑니다. 링크를 외부에 공유하지 마세요. (운영 단계에선 Apps Script API로 보호하는 것을 권장)

---

## 5. Apps Script 백엔드 (자동화 · 키 숨김)

**구글시트 → 확장 프로그램 → Apps Script** 에 붙여넣고, `OPENAI_KEY`를
스크립트 속성(프로젝트 설정 → 스크립트 속성)에 저장하세요. (코드에 직접 쓰지 말 것)

```javascript
// 새 응답 또는 logo_status=접수 행을 찾아 AI 로고 분석 생성 → 시트 업데이트 → 이메일 발송
function processNewApplicants() {
  const key = PropertiesService.getScriptProperties().getProperty('OPENAI_KEY');
  const sh = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
  const data = sh.getDataRange().getValues();
  const head = data[0];
  const idx = name => head.indexOf(name);

  for (let r = 1; r < data.length; r++) {
    const row = data[r];
    const status = row[idx('logo_status')];
    if (status && status !== '접수' && status !== '') continue; // 미처리만

    const nick = row[idx('nickname')] || '';
    const color = row[idx('favorite_color')] || '';
    const vals = [row[idx('value1')], row[idx('value2')], row[idx('value3')]].filter(String).join(', ');
    const symbol = row[idx('symbol')] || '';
    const proud = row[idx('proud_moment')] || '';
    const challenge = row[idx('challenge')] || '';
    const email = row[idx('email')] || '';
    if (!email) continue;

    sh.getRange(r + 1, idx('logo_status') + 1).setValue('생성중');

    const prompt =
      '너는 세계 최고의 브랜드 전략가이자 로고 디자이너다. 아래 고객의 정체성과 인생 서사가 담긴 아이덴티티 로고를 설계하라.\n' +
      '고객: 애칭=' + nick + ' / 색=' + color + ' / 가치관=' + vals + ' / 상징=' + symbol +
      ' / 자랑스러운순간=' + proud + ' / 극복한어려움=' + challenge + '\n' +
      'JSON만 출력(한국어). {"identity":"300자 이내 감성 분석","keywords":["키워드3"],' +
      '"slogan_ko":["한글3"],"slogan_en":["영어3"],' +
      '"image_prompt":"영어. minimal premium vector logo, clean geometry, symbolic, emotional, suitable for phone case, works in black and white, no mockup, no text, luxury brand identity style",' +
      '"case_apply":"케이스 적용 설명"}';

    const res = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', {
      method: 'post', contentType: 'application/json',
      headers: { Authorization: 'Bearer ' + key },
      payload: JSON.stringify({ model: 'gpt-4o-mini', temperature: 0.8,
        messages: [{ role: 'user', content: prompt }] }),
      muteHttpExceptions: true
    });
    let out = {};
    try {
      const txt = JSON.parse(res.getContentText()).choices[0].message.content
        .replace(/```json/gi, '').replace(/```/g, '').trim();
      out = JSON.parse(txt.slice(txt.indexOf('{'), txt.lastIndexOf('}') + 1));
    } catch (e) {
      sh.getRange(r + 1, idx('logo_status') + 1).setValue('접수'); // 실패 → 재시도 대상
      continue;
    }

    sh.getRange(r + 1, idx('generated_identity') + 1).setValue(out.identity || '');
    sh.getRange(r + 1, idx('generated_slogan') + 1).setValue(
      [].concat(out.slogan_ko || [], out.slogan_en || []).join(' / '));
    sh.getRange(r + 1, idx('image_prompt') + 1).setValue(out.image_prompt || '');
    sh.getRange(r + 1, idx('logo_status') + 1).setValue('완료');

    // 이메일 발송 (마케팅 동의 무관: 신청 결과는 정보성)
    if (email) {
      MailApp.sendEmail({
        to: email,
        subject: '[SI Story] 당신의 아이덴티티 로고가 완성됐어요',
        body: '안녕하세요 ' + nick + '님,\n\n' +
              '당신의 이야기를 담은 로고 분석이 도착했습니다.\n\n' +
              '■ 아이덴티티\n' + (out.identity || '') + '\n\n' +
              '■ 슬로건\n' + [].concat(out.slogan_ko || [], out.slogan_en || []).join(' / ') + '\n\n' +
              '나만의 로고 케이스로 만들기: (스마트스토어 링크)\n\nSI Story 드림'
      });
      sh.getRange(r + 1, idx('logo_status') + 1).setValue('발송완료');
    }
  }
}

// 트리거: 시간 기반(예: 10분마다) 또는 폼 제출 시 실행
function setupTrigger() {
  ScriptApp.newTrigger('processNewApplicants').timeBased().everyMinutes(10).create();
}
```

> 이미지(로고 PNG)는 Apps Script로도 가능하지만 용량/저장 이슈가 있어, **admin.html에서 버튼으로 생성·다운로드**하는 걸 권장합니다.

---

## 6. 카카오채널 메시지 5개

1. (가입 직후) "환영해요! 🌿 SI Story는 당신의 '이름'이 아니라 '이야기'를 로고로 만듭니다. 무료 신청 폼 → [링크]"
2. (폼 작성 유도) "아직 로고 신청 전이신가요? 애칭·색·가치관 3개면 충분해요. 1분이면 끝 → [폼]"
3. (완료 안내) "✨ 당신의 아이덴티티 로고가 완성됐어요. 이메일을 확인해보세요!"
4. (사은품) "사은품(그립톡/카드지갑) 발송 준비 중이에요. 배송지가 정확한지 확인 부탁드려요 🙏"
5. (케이스 제안 · Loss Aversion) "당신만의 로고, 케이스로 남기지 않으면 사라져요. 한정 제작 39,900원 → [구매]"

---

## 7. 릴스 대본 10개 (Hook / 본문 / CTA)

1. **Hook** "이름 말고, 당신의 정체성을 로고로." / 본문 "애칭·색·가치관 3개만 알려주세요. AI가 당신만의 상징을 만듭니다." / CTA "카카오채널 가입 후 폼 작성 = 무료 + 사은품"
2. **Hook** "'고목나무에 핀 꽃'이라는 사람을 로고로 만들면?" / 본문 "오래 버틴 사람. 다시 피어나는 사람. 상처가 있어도 아름다운 사람." / CTA "당신의 이야기도 로고가 됩니다."
3. **Hook** "세상에 하나뿐인 내 로고, 무료로." / 본문 "카카오 가입 → 폼 작성 → AI 로고 → 사은품 증정" / CTA "지금 신청하기"
4. **Hook** "당신을 한 단어로 표현하면?" / 본문 "그 한 단어를 AI가 상징으로 바꿔드립니다." / CTA "무료 신청 →"
5. **Hook** "MBTI 말고, 이제 '나만의 로고'." / 본문 "성격이 아니라 서사. 당신이 버텨온 이야기를 디자인." / CTA "내 로고 받기"
6. **Hook** "이 색을 고른 이유, AI는 알아요." / 본문 "좋아하는 색에는 당신의 감정이 담겨 있어요." / CTA "내 색의 의미 보기"
7. **Hook** "선물로 받은 케이스가 '내 이야기'라면?" / 본문 "이름 이니셜 말고, 당신의 상징을 새깁니다." / CTA "무료로 만들어보기"
8. **Hook** "3초면 이해되는 영상: 무료 로고 이벤트" / 본문 "①가입 ②폼 ③로고 ④사은품" / CTA "링크 클릭"
9. **Hook** "당신의 극복 서사를 로고로." / 본문 "가장 힘들었던 순간이, 가장 강한 상징이 됩니다." / CTA "내 서사 디자인하기"
10. **Hook** "무료인데 이렇게 고급스러워도 돼?" / 본문 "네이비 × 골드, 미니멀 프리미엄. 흑백으로도 살아있는 로고." / CTA "지금 무료 신청"

---

## 8. 개인정보 동의 문구

> 수집된 정보는 **AI 로고 제작, 사은품 발송, 이벤트 안내 및 맞춤 상품 제안** 목적으로 사용됩니다.
> 마케팅 수신은 **선택**이며, 동의하지 않아도 무료 로고 신청은 가능합니다.
> 주소·연락처는 **사은품 발송 목적 외 사용하지 않습니다.**

---

## 9. 보안 체크 (꼭)

- **공개 페이지(event/logo)에는 API 키 절대 넣지 않음** — 현재 코드도 키 없음.
- AI 호출은 **Apps Script(키는 스크립트 속성)** 또는 **admin.html(본인 PC)** 에서만.
- admin.html은 **검색 노출/공유 금지**. 운영 시 비공개 경로·인증 추가 권장.
- 개인정보 최소 수집, 마케팅 동의 분리, 주소/연락처 목적 외 사용 금지.

---

## 10. 개발 단계별 체크리스트

- [ ] 1. 구글폼 16개 필드 생성 → 시트 연결
- [ ] 2. 시트에 logo_status 등 추가 컬럼 생성
- [ ] 3. event.html / logo.html CONFIG 주소 교체 (카카오·폼·샵·entry)
- [ ] 4. 6+3 페이지를 GitHub(또는 호스팅)에 업로드 (case-studio-app.html → index.html)
- [ ] 5. 메인 설정에서 OpenAI 키 저장 (admin·logo생성 공유)
- [ ] 6. 시트 "웹에 게시(CSV)" → admin.html에 URL 입력
- [ ] 7. Apps Script 붙여넣기 + 스크립트 속성에 OPENAI_KEY 저장 + 트리거 설정
- [ ] 8. 테스트: 폼 제출 → 시트 저장 → admin/스크립트로 생성 → 이메일 확인
- [ ] 9. 릴스 업로드 + 카카오 메시지 세팅
- [ ] 10. 케이스 구매 링크(스마트스토어) 연결 확인
