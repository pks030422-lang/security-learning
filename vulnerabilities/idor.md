# IDOR (Insecure Direct Object Reference)

> 진행 상태: 아래에 적은 실습은 모두 완료(2026-10-15, 사용자 확인). 일부 실습은 당시 화면·HTTP 응답 전문을 보관하지 않았으므로 관찰 기록의 범위를 별도로 표시합니다.

## 한 문장으로 설명

사용자가 지정한 `id` 같은 값으로 서버가 대상을 찾을 때, **현재 로그인한 사용자에게 그 대상을 읽거나 수정할 권한이 있는지** 제대로 확인하지 않으면 생길 수 있는 접근 제어 취약점입니다. [PortSwigger 설명](https://portswigger.net/web-security/access-control/idor)

## 이해에 필요한 개념

- **Endpoint:** 클라이언트가 특정 기능을 요청하는 주소입니다. 예를 들어 `/app/updateUser`는 사용자 정보 수정을 처리하는 경로입니다.
- **HTTP request:** 브라우저가 서버에 보내는 요청입니다. URL, 헤더, 본문 등에 값이 들어갈 수 있습니다.
- **인증(Authentication):** 요청을 보낸 사람이 누구인지 확인하는 과정입니다.
- **인가(Authorization):** 그 사람이 **이 데이터에** 접근하거나 **이 작업을** 할 수 있는지 확인하는 과정입니다.
- **JSON:** 요청 본문에 데이터를 담는 형식 중 하나입니다. `{"id": 123}`처럼 키와 값으로 표현합니다.
- **Query parameter:** URL의 `?` 뒤에 붙는 `이름=값` 형식의 값입니다. `/my-account?id=wiener`에서 `id`가 그 예입니다.
- **GUID (Globally Unique Identifier):** 대상을 구별하는 긴 식별자입니다. 이번 실습에서는 사용자 이름 대신 GUID가 `id` 값으로 쓰였습니다. 값이 길고 추측하기 어려워도 접근 권한을 증명하지는 않습니다.
- **Elements / DOM:** 브라우저 개발자 도구의 Elements는 현재 페이지를 구성하는 HTML 요소와 속성을 보여줍니다. 여기서 바꾼 내용은 내 브라우저 화면에만 적용됩니다.
- **`type="password"`:** 입력칸의 값을 점으로 가려 보여주는 HTML 설정입니다. 값 자체를 암호화하는 기능은 아닙니다.

## 정상 동작과 취약한 동작

정상이라면 A 계정으로 로그인한 요청에서 대상 `id`를 B로 바꿔도, 서버가 A의 권한을 확인하고 B의 정보 열람·수정을 거부해야 합니다. IDOR이 있으면 대상 값만 바꾼 요청이 받아들여질 수 있습니다.

식별자(`id`, GUID, `BasketId` 등)는 **어떤 대상을 요청하는지** 알려줄 뿐입니다. 인증은 “지금 로그인한 사람이 A인가?”를, 인가는 “A가 이 대상에 접근해도 되는가?”를 판단합니다. 따라서 식별자가 길거나 화면에서 보이지 않아도 서버의 인가 검사가 필요합니다. 이 실습들에서 사용한 IDOR은 다른 사람의 데이터를 읽는 경우가 중심입니다. 읽기뿐 아니라 수정·삭제 요청에도 같은 원리가 적용됩니다.

## 지금까지 실습한 흐름과 각 단계의 이유

| 순서 | 실습과 진행 상태 | 실제로 한 일 | 그 단계를 한 이유 |
| --- | --- | --- | --- |
| 1 | [PortSwigger — URL의 사용자 ID](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter) · **Solved 확인** | `wiener`로 로그인하고 `/my-account?id=wiener`의 `id`를 `carlos`로 바꿔 Carlos의 정보가 보이는지 확인 | 로그인한 사람은 그대로 두고 **요청 대상만** 바꿨을 때 서버가 소유권을 확인하는지 시험 |
| 2 | [PortSwigger — GUID 사용자 ID](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids) · **완료(사용자 확인)** | Carlos 작성자 링크에서 GUID를 확인한 뒤, 로그인한 계정 페이지의 `id`에 적용 | 식별자를 추측하기 어렵더라도 다른 페이지에 노출될 수 있고, GUID 자체는 접근 권한이 아님을 확인 |
| 3 | [PortSwigger — 비밀번호 노출](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure) · **완료(사용자 확인)** | 다른 계정의 페이지에 접근하고 Elements에서 Password 입력칸의 표시 방식(`type`)을 바꿔 이미 브라우저에 전달된 값을 확인 | 화면에서 가려진 값과 서버가 데이터를 보내지 않은 상태를 구별하고, 실제 문제인 타인 계정 정보 제공을 이해 |
| 4 | [PortSwigger — `roleid` 변경](role-escalation.md) · **본인 완료 확인** | 이메일 변경 요청의 Payload를 살펴보고 요청 본문에 `roleid`를 추가해 실습 | 같은 접근 제어 주제지만, 다른 객체의 ID를 요청하는 IDOR과 **자기 계정의 역할을 올리는 권한 상승**을 비교 |
| 5 | [PortSwigger — 보호되지 않은 관리자 기능](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality) · **완료(사용자 확인)** | 개발자 도구의 폴더 표시와 `/robots.txt` 단서가 어디 있는지 질문 | 숨겨진 관리자 경로 발견과, 그 경로를 서버가 누구에게 허용하는지는 다른 문제임을 이해 |
| 6 | [OWASP Juice Shop — View Basket](https://help.owasp-juice.shop/part2/broken-access-control.html#view-another-users-shopping-basket) · **완료(사용자 확인)** | 교육용 로컬 환경에서 A/B 계정과 서로 다른 장바구니 상품을 만들고 Network 응답의 ID들을 비교 | 계정과 장바구니가 어떻게 연결되는지 알아낸 뒤, A의 로그인 상태로 B의 장바구니를 요청할 수 있는지 검증 |

**공통 실험 흐름:** 정상 기능을 한 번 사용한다 → 실제 요청과 식별자 위치를 찾는다 → 로그인 계정은 유지한 채 대상 식별자만 바꾼다 → 서버 응답과 화면 결과를 비교한다. 값을 **바꿀 수 있다는 사실만으로** 취약점이 증명되지는 않습니다. 서버가 권한 없는 대상의 데이터를 실제로 돌려주거나 변경했는지 확인해야 합니다.

## 참고한 공개 리포트

- [HackerOne #1714638: 프로필 수정 관련 IDOR](https://hackerone.com/reports/1714638)
- 읽으면서 확인할 질문: 요청의 어떤 값이 수정할 계정을 가리키는가? 로그인한 계정과 수정 대상은 어떻게 다른가? 서버가 어떤 검사를 놓쳤는가?
- **읽은 내용과 직접 검증한 결과는 구분해서 추가할 것.** 이 리포트의 원래 서비스에 요청을 보내지 않습니다.

### 입문 순서로 읽을 만한 실제 제출 사례

아래 링크는 교육용 랩 풀이가 아니라 HackerOne에 제출되어 공개된 제보입니다. HackerOne의 사례 해설도 함께 연결해, 보고서 페이지에서 설명이 부족할 때 대조할 수 있게 했습니다.

| 순서 | 제출 보고서 | 무엇을 관찰할까? | 대응하는 내 실습 |
| --- | --- | --- | --- |
| 1 | [Unikrn #1966006 — 사용자 정보 조회](https://hackerone.com/reports/1966006) · [HackerOne 사례 소개](https://www.hackerone.com/blog/pentesting-web-applications) | 다른 사용자 ID를 통해 이메일·전화번호가 노출되는 **읽기형 IDOR**. 어떤 식별자가 대상을 고르고, 응답에 무엇이 나왔는지 보기 | URL의 사용자 ID 변경, View Basket |
| 2 | [MTN #1714638 — 다른 계정 프로필 수정](https://hackerone.com/reports/1714638) · [HackerOne 단계별 해설](https://www.hackerone.com/blog/how-idor-vulnerability-led-user-profile-modification) | 공격자·피해자 계정을 분리하고, `/app/updateUser` 요청의 `id`와 `email`을 바꾼 뒤 피해자 정보가 실제로 수정됐는지 보기 | A/B 계정 구분, 요청 본문과 서버 결과 확인 |
| 3 | [Shopify #322661 — 다른 계정의 첨부 문서 교체](https://hackerone.com/reports/322661) · [HackerOne 사례 소개](https://www.hackerone.com/blog/rise-idor) | 계정마다 같은 이름으로 참조되는 문서가 있을 때 대상 계정의 문서를 바꿀 수 있었던 사례. ID가 숫자가 아닌 **파일 이름**일 수도 있다는 점 보기 | 장바구니 ID 이외의 객체 식별자 찾기 |

각 보고서에서 **① 누가 로그인했는가 ② 대상 객체는 무엇인가 ③ 어느 값을 바꿨는가 ④ 서버가 실제로 무엇을 돌려주거나 변경했는가**를 네 줄로 요약해 봅니다. 공개된 보고서는 과거의 수정된 사례입니다. 원래 서비스에 같은 요청을 다시 보내지 않습니다.

## 직접 완료한 실습: URL의 `id` 변경

- 환경: [PortSwigger Web Security Academy — User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)
- 목표: `wiener`로 로그인한 상태에서 `carlos`의 API Key를 확인하고 제출합니다.
- 결과: **Solved** 표시를 확인했습니다. 이 실습은 **정보 열람** 사례로, 위 리포트의 **프로필 수정**과는 결과가 다릅니다.

### 내가 한 일과 관찰한 결과

1. 실습 사이트에서 제공한 `wiener` 계정으로 로그인했습니다. 계정 페이지의 주소는 `/my-account?id=wiener`였고, 화면에 `wiener`의 정보가 보였습니다.
2. **같은 실습 사이트에서** 주소 끝의 `id=wiener`를 `id=carlos`로 바꿔 접속했습니다. 이때 별도로 `carlos` 계정에 로그인하지 않았습니다.
3. 화면에 `Your username is: carlos`와 Carlos의 API Key가 표시됐습니다. 상단의 `My account` 링크는 여전히 `id=wiener`를 가리켰습니다.
4. API Key를 실습의 정답으로 제출했고 화면에서 `Solved`를 확인했습니다.

### 단계마다 왜 그렇게 했나?

1. **왜 `wiener`로 먼저 로그인했나?** 서버가 나를 일반 사용자 `wiener`로 인식한 상태를 만들어야, 다른 사람의 정보가 보일 때 권한 검사 문제인지 판단할 수 있기 때문입니다. 로그인하지 않은 상태로 같은 페이지가 열리는지는 이 실습에서 확인한 결과가 아닙니다.
2. **왜 계정 페이지의 URL을 봤나?** `/my-account?id=wiener`의 `id`가 현재 표시할 사용자를 선택하는 값인지 알아보기 위해서입니다. 주소창은 사용자가 직접 바꿀 수 있는 입력 위치입니다.
3. **왜 `id=carlos`로 바꿨나?** 로그인은 `wiener`로 유지하고 **데이터 대상만 Carlos로 바꿔** 서버가 소유권을 검사하는지 시험하려고 했습니다. Carlos 계정으로 로그인했다면 타인 정보 접근 시험이 되지 않습니다.
4. **왜 화면의 사용자 이름과 API Key를 확인했나?** 주소를 바꿀 수 있다는 사실만으로 취약점이 입증되지는 않습니다. 서버가 실제로 Carlos의 데이터를 보여줬는지를 확인해야 합니다.
5. **왜 실습에 API Key를 제출했나?** 내가 본 결과가 실습 목표에 맞는지 검증하기 위해서입니다. `Solved`는 이 교육용 문제의 완료 신호이고, 원인은 앞 단계의 권한 없는 정보 열람입니다.

**기대했던 안전한 동작:** 로그인한 사용자가 다른 계정의 식별자를 넣어도 서버가 그 계정의 정보를 보여주지 않아야 합니다.

**배운 점:** URL의 `id`는 누구의 정보를 요청할지 정하는 값일 뿐, 접근 권한을 증명하지 않습니다. 서버가 로그인한 사용자와 요청 대상의 권한 관계를 확인하지 않아 다른 사용자의 정보가 노출됐습니다. 이 실습에서는 브라우저 주소창만 사용했고, Burp Suite로 HTTP request나 쿠키 값을 직접 확인하지는 않았습니다. 실제 API Key·세션 값·실습 인스턴스 주소는 기록하지 않습니다.

## 두 번째 실습: GUID로 표시된 사용자 ID

- 환경: [PortSwigger — User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)
- 목표: Carlos의 GUID를 찾아, `wiener`로 로그인한 상태에서 Carlos의 계정 정보에 접근할 수 있는지 확인합니다.
- 진행 상태: 완료(2026-10-15 사용자 확인). Carlos 작성자 링크에서 GUID를 찾아 계정 페이지의 `id`에 적용한 과정은 기록했으며, 당시 화면은 보관하지 않았습니다.

### 내가 한 과정

1. 실습 사이트에서 Carlos가 작성한 글을 찾고, **작성자 Carlos의 링크**를 열어 주소에 드러난 GUID를 확인했습니다.
2. 로그인 화면으로 돌아가 실습용 `wiener` 계정으로 로그인했습니다.
3. 내 계정 페이지 주소의 `id` 값을 Carlos의 작성자 링크에서 확인한 GUID로 바꿔 접속했습니다. GUID를 로그인 입력칸에 넣은 것이 아니라, 계정 페이지의 URL에 있는 `id` 값으로 사용했습니다.

### 단계마다 왜 그렇게 했나?

1. **왜 Carlos의 작성자 링크를 봤나?** GUID는 사용자 이름처럼 쉽게 떠올릴 수 없지만, 사이트가 글 작성자를 연결하기 위해 이미 그 값을 공개할 수 있습니다. 먼저 대상 식별자를 알아야 요청 대상을 바꿔 볼 수 있습니다.
2. **왜 그다음 `wiener`로 로그인했나?** 로그인한 계정과 요청할 계정을 다르게 유지해 인가 검사를 시험하려고 했습니다.
3. **왜 GUID를 로그인창이 아니라 URL의 `id`에 넣었나?** 로그인창은 ‘내가 누구인가’를 확인하는 곳이고, 계정 페이지의 `id`는 ‘누구의 데이터를 보여 달라는가’를 지정하는 곳입니다. 취약점의 핵심은 두 값이 다를 때 서버의 판단입니다.
4. **무엇까지 기록했나?** GUID를 찾아 적용한 과정입니다. 전체 실습은 사용자가 완료했다고 확인했지만 결과 화면과 `Solved` 배지의 별도 증거는 이 메모에 없습니다.

**배운 점:** GUID를 무작정 추측할 수 없어도, 사이트의 다른 페이지에서 공개된 식별자를 찾아 사용할 수 있습니다. GUID가 길다는 사실은 서버의 인가(Authorization) 검사를 대신하지 않습니다. 실습 인스턴스의 실제 GUID·API Key·세션 값은 기록하지 않습니다.

## 세 번째 실습: 계정 페이지의 비밀번호 노출

- 환경: [PortSwigger — User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)
- 실습 목표: `administrator`의 비밀번호를 알아내 관리자 계정으로 로그인한 뒤 Carlos를 삭제합니다.
- 진행 상태: 완료(2026-10-15 사용자 확인). 비밀번호 입력칸을 검사한 과정과 배운 점을 기록했으며, 당시 결과 화면은 보관하지 않았습니다.

### 내가 한 과정

1. 실습용 `wiener` 계정으로 로그인하고 계정 페이지의 `id`를 다른 사용자로 바꿔 보았습니다. 처음에는 Carlos 계정의 Password 칸이 점으로 가려진 것을 확인했습니다.
2. 실습 목표가 Carlos의 비밀번호가 아니라 **administrator의 비밀번호**라는 것을 확인하고, 주소의 `id`를 `administrator`로 바꿨습니다.
3. Password 칸을 오른쪽 클릭해 검사(Inspect)를 열었습니다. Elements에서 해당 입력칸의 `type="password"`를 `type="text"`로 바꿔, 브라우저에 이미 들어 있던 값을 글자로 확인했습니다. 실제 비밀번호는 기록하지 않습니다.

### 단계마다 왜 그렇게 했나?

1. **왜 다른 사용자의 계정 페이지를 열었나?** `wiener`로 로그인한 상태에서도 다른 계정의 정보가 전달되는지 확인하려는 IDOR 실험이었습니다.
2. **왜 Carlos에서 administrator로 대상을 바꿨나?** 실습의 실제 목표가 administrator 계정의 비밀번호를 이용하는 것이기 때문입니다. Carlos의 값을 알아낸 것만으로는 이 문제의 목표에 도달하지 않습니다.
3. **왜 Password 칸에서 검사(Inspect) → Elements를 열었나?** 점으로 가려진 입력칸의 HTML 속성과 현재 브라우저에 있는 값을 확인하려고 했습니다. Elements는 브라우저가 받은 페이지의 요소를 보여줍니다.
4. **왜 `type="password"`를 `type="text"`로 바꿨나?** 입력칸의 **표시 방식만** 바꾸면 이미 들어 있는 값을 읽을 수 있기 때문입니다. 서버 코드나 저장된 비밀번호를 바꾼 것도, 암호문을 복호화한 것도 아닙니다.
5. **왜 이게 IDOR과 연결되나?** 표시 변경은 마지막 관찰 수단일 뿐입니다. 먼저 서버가 `wiener`의 요청에 administrator의 계정 정보를 전달했다는 점이 권한 검사 실패의 핵심입니다. 전체 실습은 사용자가 완료했다고 확인했으나 관리자 로그인·계정 삭제 화면은 이 메모에 보관하지 않았습니다.

### 내가 했던 질문과 알게 된 점

- **왜 Elements를 열었나?** Password 입력칸이 어떤 HTML 요소이고, 값을 어떻게 표시하도록 설정됐는지 확인하기 위해서입니다.
- **코드를 수정해서 비밀번호를 찾은 건가?** 내 브라우저의 DOM에서 입력칸의 `type` 속성만 바꿨습니다. 서버의 코드나 계정 데이터는 수정하지 않았고, 새로고침하면 이 표시 변경은 사라집니다.
- **점으로 보이는 비밀번호는 암호화된 건가?** 아닙니다. `type="password"`가 화면에서 문자를 가려 보여준 것입니다. `type="text"`로 바꾸는 것은 복호화가 아닙니다.
- **“이미 전달된 값”이란?** 이 실습에서는 서버가 관리자 계정 페이지의 비밀번호 값을 브라우저에 보냈습니다. Elements에서 표시 방식을 바꾸자 브라우저가 가진 값이 보였습니다. 검사 기능 자체가 서버의 권한 검사를 우회한 것은 아닙니다.
- **진짜 문제는 무엇인가?** `wiener`로 로그인한 요청에 서버가 administrator의 계정 정보와 비밀번호를 제공한 접근 제어 문제입니다. 이 화면만으로 서버가 비밀번호를 어떤 방식으로 저장하는지는 알 수 없습니다.

### 이어서 진행한 허가된 실습

- [User role can be modified in user profile](role-escalation.md) — 이메일 변경 요청 본문에 `roleid`를 추가해 권한 상승을 확인한 별도 기록입니다. 이는 다른 계정의 객체 식별자를 바꾸는 IDOR과 구분합니다.

## 네 번째 IDOR 실습: OWASP Juice Shop의 다른 사람 장바구니 보기

- 환경: 내 컴퓨터에서 실행한 교육용 [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/). 문제 설명: [View another user's shopping basket](https://help.owasp-juice.shop/part2/broken-access-control.html#view-another-users-shopping-basket).
- 목표: **A로 로그인한 상태에서** B의 장바구니 내용에 접근되는지 확인합니다.
- 현재 상태: 완료(2026-10-15 사용자 확인). 아래 A/B 값은 대화에서 직접 확인했으며, `bid` 변경 뒤의 당시 Request·Response와 완료 알림 화면은 별도로 보관하지 않았습니다.

### 내가 실제로 관찰한 값

| 계정 | User ID | Basket ID | BasketItem `id` | Product ID | 수량 |
| --- | ---: | ---: | ---: | ---: | ---: |
| A | 25 | 6 | 10 | 1 | 1 |
| B | 26 | 7 | 11 | 24 | 1 |

이 숫자는 당시 **로컬 실습 인스턴스**에서 본 값이라 새로 실행하거나 계정을 다시 만들면 달라질 수 있습니다. `User ID`는 계정, `Basket ID`는 장바구니, `BasketItem.id`는 장바구니의 개별 항목, `Product ID`는 상품을 가리킵니다. View Basket에서 바꿔 볼 대상은 **Basket ID**입니다. B의 장바구니 안에 A와 다른 상품이 있어야, 결과 화면을 보고 어느 장바구니가 열렸는지 구분할 수 있습니다.

### 왜 Network → Fetch/XHR → Application → Session Storage 순서로 보았나?

1. **두 계정으로 각자 상품 담기:** A와 B의 장바구니 내용을 다르게 만들어 비교 기준을 마련했습니다.
2. **Chrome 개발자 도구의 Network 열기:** 주소창과 화면만으로는 브라우저가 서버에 어떤 장바구니를 요청하는지 모두 보이지 않기 때문입니다. Network는 브라우저와 서버 사이의 HTTP 요청·응답을 보여줍니다.
3. **Fetch/XHR 필터 선택:** JavaScript가 화면 뒤에서 보낸 데이터 요청에 집중하려고 목록을 좁혔습니다. 이 필터는 요청을 수정하거나 새로 보내지 않습니다. 원하는 요청이 없으면 **All**에서도 찾습니다. Network 목록의 `6`이나 `7`은 로컬 파일 이름이라는 뜻이 아닙니다. 항목을 클릭하고 **Headers → Request URL**을 봐야 전체 요청 주소를 확인할 수 있습니다.
4. **장바구니를 열고 관련 요청의 Preview/Response 확인:** 응답에 `BasketId`와 상품 정보가 있는지 보아 어떤 요청이 장바구니 데이터를 가져오는지 구별했습니다. Request URL은 **무엇을 요청했는지**, Response는 **서버가 실제로 무엇을 돌려줬는지** 알려줍니다. IDOR 판단에는 둘 다 필요합니다. Response에 보인 JSON을 편집해도 서버 데이터가 바뀌지는 않습니다.
5. **Application → Session Storage 확인:** 이 실습의 브라우저 앱이 장바구니 선택에 쓰는 `bid` 값을 어디에 보관하는지 확인하기 위한 단계입니다. `bid`는 *basket ID*를 뜻하는 앱의 저장 키입니다. 주소창의 `/#/basket`는 **장바구니 화면 경로**이고, 선택할 장바구니 번호를 반드시 URL에 표시하지는 않습니다. 앱은 저장된 `bid`를 읽어 데이터 요청을 만들 수 있습니다. **Network는 실제 요청을 관찰하는 곳, Session Storage는 이 앱이 요청에 쓰는 값을 보관하는 곳**입니다.

### 지금까지 했던 질문

- **왜 Fetch/XHR을 골랐나?** HTML·CSS·이미지 등 여러 요청 중 장바구니 데이터를 주고받는 요청을 빨리 찾기 위해서입니다. 선택하지 않아도 요청 자체는 같습니다.
- **Network에서 어떤 “파일”을 찾나?** 파일을 찾는 것이 아니라, 장바구니를 열 때 발생한 **HTTP 요청**을 찾습니다. `Name`이 숫자여도 Headers의 전체 Request URL과 Preview/Response의 `BasketId`를 함께 확인합니다.
- **왜 응답을 보나?** 요청 주소만 봐서는 서버가 다른 사람의 데이터를 줬는지 알 수 없습니다. 응답의 상품과 `BasketId`가 결과 증거입니다.
- **`bid`가 뭔가?** 브라우저 앱이 참조하는 *basket ID* 저장 키입니다. A의 장바구니 ID가 `6`, B의 ID가 `7`인 것과 연결해 생각합니다. `BasketItem.id`나 `ProductId`와는 다른 값입니다.
- **URL에 `bid`가 없는데 왜 Session Storage를 보나?** 주소창의 화면 경로와 앱 내부의 장바구니 선택 값은 별개일 수 있습니다. 저장소 값을 읽어 서버 요청을 만들 수 있어서, 이 앱의 `bid`가 있는지 확인합니다.
- **Session Storage가 서버의 장바구니 데이터를 바꾸나?** 아닙니다. 내 브라우저 탭에 보관된 값을 바꾸는 것입니다. 이후 앱이 그 값으로 새 요청을 보냈을 때 서버가 허용하는지가 핵심입니다.

### 실습의 마지막 단계가 왜 필요한가?

1. **A로 로그인한 탭**에서 Application → Session Storage → 실습 사이트의 `bid`를 확인하는 이유: A의 Basket ID인 `6`과 앱이 저장한 값이 연결되는지 알아보기 위해서입니다.
2. 교육용 로컬 실습에서만 `bid`를 B의 Basket ID인 `7`로 바꿔 장바구니 화면을 다시 여는 이유: 로그인 계정은 A로 유지하고 요청 대상 장바구니만 B로 바꿔 서버의 소유권 검사를 시험하기 위해서입니다.
3. 새 요청의 Response와 화면에서 B의 상품(`ProductId: 24`) 및 완료 알림을 확인하는 이유: 저장소 값을 수정할 수 있다는 사실만으로는 IDOR이 증명되지 않으며, 서버가 실제로 B의 데이터를 제공했는지 봐야 하기 때문입니다.
4. 예상과 다른 결과가 나왔을 때 **Request URL, 상태 코드, 응답의 `BasketId`**를 다시 보는 이유: 저장 값만 바뀌고 실제 요청은 달라지지 않았을 수도 있기 때문입니다.

이 절차는 [OWASP Juice Shop 공식 풀이](https://help.owasp-juice.shop/appendix/solutions.html#view-another-users-shopping-basket)의 `bid` 확인·변경 흐름을 참고했습니다. 사용자가 실습 완료를 확인했으며, 마지막 요청·응답의 정확한 값은 대화에 남아 있지 않아 새로 만들어 기록하지 않습니다.

## 원인과 대응

핵심은 서버가 클라이언트가 보낸 대상 식별자를 그대로 신뢰하는지 여부입니다. 수정 기능이라면 서버는 로그인한 사용자에게 **해당 대상의 수정 권한**이 있는지 요청마다 확인해야 합니다. 단순히 `id`를 숨기거나 추측하기 어렵게 만드는 것만으로는 권한 검사를 대신할 수 없습니다.

## 다음에 확인할 질문

- 같은 실습을 Burp Suite의 HTTP history와 Repeater로 다시 해보면 요청의 URL, 헤더, 쿠키는 각각 어디에 보이는가?
- URL의 `id`를 바꾸는 실습과 JSON 본문의 `id`를 바꾸는 사례는 무엇이 같은가?
- 서버가 거부해야 할 요청은 어떤 응답으로 나타나는가?
