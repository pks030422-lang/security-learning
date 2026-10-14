# IDOR (Insecure Direct Object Reference)

> 진행 상태: 첫 실습 완료 (2026-10-14). 브라우저 주소창에서 `id`를 바꿔 결과를 확인했습니다.

## 한 문장으로 설명

사용자가 지정한 `id` 같은 값으로 서버가 대상을 찾을 때, **현재 로그인한 사용자에게 그 대상을 읽거나 수정할 권한이 있는지** 제대로 확인하지 않으면 생길 수 있는 접근 제어 취약점입니다. [PortSwigger 설명](https://portswigger.net/web-security/access-control/idor)

## 이해에 필요한 개념

- **Endpoint:** 클라이언트가 특정 기능을 요청하는 주소입니다. 예를 들어 `/app/updateUser`는 사용자 정보 수정을 처리하는 경로입니다.
- **HTTP request:** 브라우저가 서버에 보내는 요청입니다. URL, 헤더, 본문 등에 값이 들어갈 수 있습니다.
- **인증(Authentication):** 요청을 보낸 사람이 누구인지 확인하는 과정입니다.
- **인가(Authorization):** 그 사람이 **이 데이터에** 접근하거나 **이 작업을** 할 수 있는지 확인하는 과정입니다.
- **JSON:** 요청 본문에 데이터를 담는 형식 중 하나입니다. `{"id": 123}`처럼 키와 값으로 표현합니다.
- **Query parameter:** URL의 `?` 뒤에 붙는 `이름=값` 형식의 값입니다. `/my-account?id=wiener`에서 `id`가 그 예입니다.

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

### 다음 허가된 실습 (아직 시도하지 않음)

1. [User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids) — 다른 사람의 ID가 GUID처럼 길어도, 사이트 안에서 그 값을 찾은 뒤 권한 검사를 확인합니다.
2. [User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure) — 비슷한 URL 변경이 더 민감한 정보 노출로 이어지는 사례를 살펴봅니다. Burp로 응답을 읽는 연습이 필요합니다.
3. [User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile) — 주소창만으로는 보이지 않는 JSON 요청 본문을 Burp로 수정하는 연습입니다. 현재 실습보다 한 단계 어렵습니다.

## 원인과 대응

핵심은 서버가 클라이언트가 보낸 대상 식별자를 그대로 신뢰하는지 여부입니다. 수정 기능이라면 서버는 로그인한 사용자에게 **해당 대상의 수정 권한**이 있는지 요청마다 확인해야 합니다. 단순히 `id`를 숨기거나 추측하기 어렵게 만드는 것만으로는 권한 검사를 대신할 수 없습니다.

## 다음에 확인할 질문

- 같은 실습을 Burp Suite의 HTTP history와 Repeater로 다시 해보면 요청의 URL, 헤더, 쿠키는 각각 어디에 보이는가?
- URL의 `id`를 바꾸는 실습과 JSON 본문의 `id`를 바꾸는 사례는 무엇이 같은가?
- 서버가 거부해야 할 요청은 어떤 응답으로 나타나는가?
