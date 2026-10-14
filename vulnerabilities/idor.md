# IDOR (Insecure Direct Object Reference)

> 진행 상태: 첫 실습 완료, GUID·비밀번호 노출 실습 과정 기록 (2026-10-14).

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

## 참고한 공개 리포트

- [HackerOne #1714638: 프로필 수정 관련 IDOR](https://hackerone.com/reports/1714638)
- 읽으면서 확인할 질문: 요청의 어떤 값이 수정할 계정을 가리키는가? 로그인한 계정과 수정 대상은 어떻게 다른가? 서버가 어떤 검사를 놓쳤는가?
- **읽은 내용과 직접 검증한 결과는 구분해서 추가할 것.** 이 리포트의 원래 서비스에 요청을 보내지 않습니다.

## 직접 완료한 실습: URL의 `id` 변경

- 환경: [PortSwigger Web Security Academy — User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)
- 목표: `wiener`로 로그인한 상태에서 `carlos`의 API Key를 확인하고 제출합니다.
- 결과: **Solved** 표시를 확인했습니다. 이 실습은 **정보 열람** 사례로, 위 리포트의 **프로필 수정**과는 결과가 다릅니다.

### 내가 한 일과 관찰한 결과

1. 실습 사이트에서 제공한 `wiener` 계정으로 로그인했습니다. 계정 페이지의 주소는 `/my-account?id=wiener`였고, 화면에 `wiener`의 정보가 보였습니다.
2. **같은 실습 사이트에서** 주소 끝의 `id=wiener`를 `id=carlos`로 바꿔 접속했습니다. 이때 별도로 `carlos` 계정에 로그인하지 않았습니다.
3. 화면에 `Your username is: carlos`와 Carlos의 API Key가 표시됐습니다. 상단의 `My account` 링크는 여전히 `id=wiener`를 가리켰습니다.
4. API Key를 실습의 정답으로 제출했고 화면에서 `Solved`를 확인했습니다.

**기대했던 안전한 동작:** 로그인한 사용자가 다른 계정의 식별자를 넣어도 서버가 그 계정의 정보를 보여주지 않아야 합니다.

**배운 점:** URL의 `id`는 누구의 정보를 요청할지 정하는 값일 뿐, 접근 권한을 증명하지 않습니다. 서버가 로그인한 사용자와 요청 대상의 권한 관계를 확인하지 않아 다른 사용자의 정보가 노출됐습니다. 이 실습에서는 브라우저 주소창만 사용했고, Burp Suite로 HTTP request나 쿠키 값을 직접 확인하지는 않았습니다. 실제 API Key·세션 값·실습 인스턴스 주소는 기록하지 않습니다.

## 두 번째 실습: GUID로 표시된 사용자 ID

- 환경: [PortSwigger — User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)
- 목표: Carlos의 GUID를 찾아, `wiener`로 로그인한 상태에서 Carlos의 계정 정보에 접근할 수 있는지 확인합니다.
- 진행 상태: Carlos 작성자 링크에서 GUID를 찾아 계정 페이지의 `id`에 적용했습니다. 화면의 `Solved` 표시 여부는 아직 확인하지 못했습니다.

### 내가 한 과정

1. 실습 사이트에서 Carlos가 작성한 글을 찾고, **작성자 Carlos의 링크**를 열어 주소에 드러난 GUID를 확인했습니다.
2. 로그인 화면으로 돌아가 실습용 `wiener` 계정으로 로그인했습니다.
3. 내 계정 페이지 주소의 `id` 값을 Carlos의 작성자 링크에서 확인한 GUID로 바꿔 접속했습니다. GUID를 로그인 입력칸에 넣은 것이 아니라, 계정 페이지의 URL에 있는 `id` 값으로 사용했습니다.

**배운 점:** GUID를 무작정 추측할 수 없어도, 사이트의 다른 페이지에서 공개된 식별자를 찾아 사용할 수 있습니다. GUID가 길다는 사실은 서버의 인가(Authorization) 검사를 대신하지 않습니다. 실습 인스턴스의 실제 GUID·API Key·세션 값은 기록하지 않습니다.

## 세 번째 실습: 계정 페이지의 비밀번호 노출

- 환경: [PortSwigger — User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)
- 실습 목표: `administrator`의 비밀번호를 알아내 관리자 계정으로 로그인한 뒤 Carlos를 삭제합니다.
- 진행 상태: 비밀번호 입력칸을 검사한 과정과 배운 점을 기록했습니다. `Solved` 표시 여부는 아직 확인하지 못했습니다.

### 내가 한 과정

1. 실습용 `wiener` 계정으로 로그인하고 계정 페이지의 `id`를 다른 사용자로 바꿔 보았습니다. 처음에는 Carlos 계정의 Password 칸이 점으로 가려진 것을 확인했습니다.
2. 실습 목표가 Carlos의 비밀번호가 아니라 **administrator의 비밀번호**라는 것을 확인하고, 주소의 `id`를 `administrator`로 바꿨습니다.
3. Password 칸을 오른쪽 클릭해 검사(Inspect)를 열었습니다. Elements에서 해당 입력칸의 `type="password"`를 `type="text"`로 바꿔, 브라우저에 이미 들어 있던 값을 글자로 확인했습니다. 실제 비밀번호는 기록하지 않습니다.

### 내가 했던 질문과 알게 된 점

- **왜 Elements를 열었나?** Password 입력칸이 어떤 HTML 요소이고, 값을 어떻게 표시하도록 설정됐는지 확인하기 위해서입니다.
- **코드를 수정해서 비밀번호를 찾은 건가?** 내 브라우저의 DOM에서 입력칸의 `type` 속성만 바꿨습니다. 서버의 코드나 계정 데이터는 수정하지 않았고, 새로고침하면 이 표시 변경은 사라집니다.
- **점으로 보이는 비밀번호는 암호화된 건가?** 아닙니다. `type="password"`가 화면에서 문자를 가려 보여준 것입니다. `type="text"`로 바꾸는 것은 복호화가 아닙니다.
- **“이미 전달된 값”이란?** 이 실습에서는 서버가 관리자 계정 페이지의 비밀번호 값을 브라우저에 보냈습니다. Elements에서 표시 방식을 바꾸자 브라우저가 가진 값이 보였습니다. 검사 기능 자체가 서버의 권한 검사를 우회한 것은 아닙니다.
- **진짜 문제는 무엇인가?** `wiener`로 로그인한 요청에 서버가 administrator의 계정 정보와 비밀번호를 제공한 접근 제어 문제입니다. 이 화면만으로 서버가 비밀번호를 어떤 방식으로 저장하는지는 알 수 없습니다.

### 다음 허가된 실습 (아직 시도하지 않음)

- [User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile) — 주소창만으로는 보이지 않는 JSON 요청 본문을 Burp로 수정하는 연습입니다. 현재 실습보다 한 단계 어렵습니다.

## 원인과 대응

핵심은 서버가 클라이언트가 보낸 대상 식별자를 그대로 신뢰하는지 여부입니다. 수정 기능이라면 서버는 로그인한 사용자에게 **해당 대상의 수정 권한**이 있는지 요청마다 확인해야 합니다. 단순히 `id`를 숨기거나 추측하기 어렵게 만드는 것만으로는 권한 검사를 대신할 수 없습니다.

## 다음에 확인할 질문

- 같은 실습을 Burp Suite의 HTTP history와 Repeater로 다시 해보면 요청의 URL, 헤더, 쿠키는 각각 어디에 보이는가?
- URL의 `id`를 바꾸는 실습과 JSON 본문의 `id`를 바꾸는 사례는 무엇이 같은가?
- 서버가 거부해야 할 요청은 어떤 응답으로 나타나는가?
