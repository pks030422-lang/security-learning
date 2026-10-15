# 요청 본문을 통한 권한 상승: `roleid` 변경

> 진행 상태: [PortSwigger 실습](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile) 완료(2026-10-14, 본인 확인). 아래에는 직접 관찰한 내용과 학습하며 이해한 내용을 구분해 적습니다. 실습 인스턴스 주소와 세션 값은 기록하지 않습니다.

## 한 문장으로 설명

이메일만 바꿔야 할 요청에 `roleid`를 추가했을 때 서버가 그 값까지 계정에 적용하면, 일반 사용자가 자신의 권한을 높일 수 있습니다. 이 실습에서 `roleid: 2`는 관리자 역할입니다. 이는 [IDOR](idor.md)처럼 다른 계정의 `id`를 바꾸는 사례와 다릅니다.

## 실습 목표와 내가 한 일

- 환경: [PortSwigger Web Security Academy — User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)
- 목표: 일반 사용자로 로그인해 관리자 역할을 얻고 `/admin`에 접근하여 실습의 지정 계정을 삭제하기
- 결과: 실습 전체 완료(본인 확인). 이 글에는 계정 삭제 화면이나 `Solved` 배지의 별도 증거를 보관하지 않았습니다.

1. 계정 페이지에서 이메일 변경 기능을 사용하고 Chrome 개발자 도구의 **Network**에서 `POST /my-account/change-email` 요청을 찾았습니다.
2. **Payload**에서 요청 본문에 `email`이 들어간다는 것을 확인했습니다. 이 요청의 본문이 바꿔 보아야 할 위치입니다.
3. 선택한 요청의 **Response**를 열었지만 Chrome은 `요청이 리디렉션되었기 때문에 사용 가능한 콘텐츠가 없습니다`라고 표시했습니다. **Headers**에서 `302 Found`와 `Location: /my-account`를 확인했습니다.
4. Chrome의 실제 우클릭 메뉴에는 `Edit and resend as fetch`가 없었습니다. 대신 **복사 → fetch로 복사**가 있어, 기존 요청을 다시 보낼 수 있는 JavaScript 코드로 복사했습니다.
5. 복사한 코드의 `body`에 `roleid`를 추가하는 방법을 익혔습니다. 원래 JSON 문자열을 직접 고치거나 `JSON.stringify`로 같은 본문을 만들 수 있습니다.
6. 사용자 확인에 따르면 실습 목표까지 완료했습니다. 요청 실행 시점의 응답 전문은 이 기록에 보관하지 않았습니다.

### 단계마다 왜 그렇게 했나?

1. **왜 먼저 이메일 변경 기능을 사용했나?** 사이트가 정상적으로 보내는 요청을 하나 만들기 위해서입니다. 정상 요청을 보아야 주소, 메서드, 본문 형식을 추측하지 않고 알 수 있습니다.
2. **왜 Network에서 변경 요청을 찾았나?** 화면의 이메일 입력칸은 서버에 어떤 HTTP 요청이 전달되는지 전부 보여주지 않습니다. `POST /my-account/change-email`을 찾으면 실제 전송 위치를 확인할 수 있습니다.
3. **왜 Payload의 `email`을 봤나?** 바꾸려는 값이 URL이 아니라 **요청 본문**에 담기는지 확인하기 위해서입니다. `roleid`를 시험하려면 이 본문에 필드를 추가해야 합니다.
4. **왜 Response와 `302`를 확인했나?** 서버가 요청을 어떻게 처리했는지 판단하려고 했습니다. `302`는 다른 페이지로 이동하라는 응답이므로 본문이 Chrome에 보이지 않을 수 있습니다. 본문이 안 보인다고 변경 성공이나 실패를 단정할 수 없습니다.
5. **왜 `fetch로 복사`한 뒤 본문을 수정했나?** Chrome에서 관찰한 요청의 주소·메서드 등을 재사용해 `roleid`만 추가한 요청을 다시 보내려는 목적입니다. 복사는 준비 단계이고 Console에서 실행해야 전송됩니다. `JSON.stringify`는 본문 문자열을 만들기 편하게 해 줄 뿐 필수는 아닙니다.
6. **왜 마지막에 관리자 기능을 확인했나?** 요청에 `roleid`를 넣을 수 있어도 서버가 무시할 수 있습니다. 실제 권한 변화와 실습 목표 달성을 확인해야 취약점의 영향을 말할 수 있습니다. 사용자가 전체 완료를 확인했으나, 당시 응답 전문은 저장하지 않았습니다.

## 이해에 필요한 개념

| 개념 | 이 실습에서의 의미 |
| --- | --- |
| 브라우저 / 서버 | 브라우저는 요청을 만들고 보내며, 서버는 로그인 상태와 권한을 판단하고 계정 정보를 처리합니다. |
| Endpoint | 특정 기능을 처리하는 주소입니다. `/my-account/change-email`은 이메일 변경 요청을 받습니다. |
| HTTP request / response | Request는 브라우저가 서버에 보내는 내용, response는 서버가 돌려주는 결과입니다. |
| `POST` | 이 실습에서 계정 정보 변경 데이터를 보내는 HTTP 메서드입니다. |
| Payload / body | 요청에 실린 데이터입니다. 처음에는 `email`이 있고, 시험할 때 `roleid`를 추가합니다. |
| JSON | `{"email":"example@example.com","roleid":2}`처럼 키와 값으로 나타내는 데이터 형식입니다. 숫자 `2`에는 따옴표가 필요 없습니다. |
| `302 Found` / `Location` | 서버가 브라우저에게 다른 주소로 이동하라고 알린 것입니다. 여기서는 `/my-account`로 이동시켰습니다. |
| 쿠키 / 세션 | 로그인한 사용자를 서버가 식별하는 데 사용합니다. 복사한 `fetch` 코드의 `credentials: "include"`는 쿠키 전송과 관련됩니다. |
| 인증 / 인가 | 인증은 '누구인가', 인가는 '이 계정이 역할을 바꿔도 되는가'를 확인하는 일입니다. |
| Proxy / Repeater | Proxy는 브라우저와 서버 사이의 요청·응답을 관찰하고 가로챌 수 있고, Repeater는 요청을 고쳐 다시 보내 비교하는 Burp 도구입니다. |

**본문 형식과 헤더는 구별해야 합니다.** 실제 Chrome 요청에서는 본문이 JSON 형태의 텍스트였지만 `Content-Type` 헤더는 `text/plain;charset=UTF-8`로 표시됐습니다. 본문에 JSON처럼 보이는 글자가 있다고 해서 헤더까지 자동으로 `application/json`이 되는 것은 아닙니다.

## 공부하며 했던 질문

### 왜 Payload를 보았나?

이메일 변경 기능이 어떤 주소에 어떤 형식으로 값을 보내는지 확인하려고 보았습니다. 화면의 이메일 입력칸에는 `roleid`가 없어도, 브라우저가 서버로 보내는 요청에는 새로운 필드를 추가해 시험할 수 있습니다.

### 왜 Response를 보려고 했나? 302는 무엇인가?

[공식 풀이](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)는 응답에서 현재 `roleid`를 발견하고 변경 결과도 확인합니다. 내 Chrome에서는 서버가 `302 Found`로 `/my-account` 이동을 지시한 요청의 본문을 Network가 보여주지 않았습니다. **응답 본문을 못 본 것이 요청 실패를 뜻하지는 않습니다.** 최종적으로 관리자 페이지에 접근할 수 있는지 확인할 수도 있습니다.

### `fetch`는 무엇이고 왜 복사했나?

`fetch()`는 JavaScript에서 HTTP 요청을 보내는 기능입니다. Chrome의 **복사 → fetch로 복사**는 이미 관찰한 요청을 다시 만들 수 있는 코드를 클립보드에 넣습니다. **복사만으로 전송되지는 않고**, Console에서 코드를 실행해야 전송됩니다. 원래 페이지가 반드시 `fetch()`를 썼다는 뜻은 아닙니다. 복사하면 주소, 메서드, 헤더 등을 직접 다시 적는 실수를 줄일 수 있습니다. [Chrome 개발자 도구 설명](https://developer.chrome.com/docs/devtools/network/reference)

### 원문에는 `JSON.stringify`가 없는데 왜 사용했나?

필수 기능이 아닙니다. 다음 두 `body` 표현은 같은 본문을 만들 수 있습니다.

```javascript
body: "{\"email\":\"example@example.com\",\"roleid\":2}"
body: JSON.stringify({email: "example@example.com", roleid: 2})
```

첫 번째는 JSON **문자열**을 직접 쓰므로 내부 따옴표 앞에 `\`가 필요합니다. 두 번째는 JavaScript **객체**를 JSON 문자열로 변환합니다. 직접 문자열을 수정할 때는 `email`과 `roleid` 사이의 쉼표가 필요하고, `2`에 `\2\`처럼 역슬래시를 붙이면 안 됩니다.

### 왜 `/my-account/admin`이 아니고 `/admin`인가?

`/admin`은 사이트의 **루트부터 시작하는 경로**입니다. 현재 페이지 주소 `/my-account` 뒤에 `admin`을 이어 붙이면 다른 주소인 `/my-account/admin`이 됩니다. 이 실습의 관리자 페이지는 `/admin`입니다.

### 왜 이런 방식으로 취약점을 찾나?

웹페이지의 입력칸은 사용자가 편하게 기능을 쓰도록 만든 UI입니다. 사용자는 자신의 브라우저가 보내는 요청 자체를 바꿀 수 있으므로, 서버는 '화면에 `roleid` 입력칸이 없다'는 이유만으로 안전하다고 가정하면 안 됩니다. 정상 요청을 먼저 관찰하고 → 보안에 중요한 필드 하나를 바꿔 보는 가설을 세우고 → 수정한 요청을 전송하고 → 실제 권한 변화로 검증합니다. **요청에 `roleid`를 적을 수 있다는 사실만으로 취약점이 증명되는 것은 아닙니다.** 서버가 값을 적용했는지가 핵심입니다.

서버 내부 코드를 보지 않았으므로 구현 원인을 단정할 수는 없습니다. 허용하지 말아야 할 필드를 일괄 적용하는 *mass assignment*와 비슷한 현상일 수 있고, 확인된 보안 영향은 **접근 제어 실패에 따른 권한 상승**입니다.

## Console 이외에 같은 요청을 시험하는 방법

- **Burp Repeater (추천):** Burp Proxy의 HTTP history에서 이메일 변경 요청을 Repeater로 보내고, 본문을 고쳐 **Send**합니다. 요청과 응답을 나란히 보고 여러 번 비교하기 쉽습니다. 이 실습의 [공식 풀이](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)도 Repeater를 사용합니다. [Burp Repeater 안내](https://portswigger.net/burp/documentation/desktop/tools/repeater)
- **Burp Intercept:** 이메일 변경 버튼을 누르기 직전에 요청을 멈춰 본문을 수정한 후 서버로 전달합니다. 첫 실험보다 재시도와 비교가 어려울 수 있습니다.
- **Chrome의 cURL로 복사:** Network의 요청을 cURL 명령으로 복사해 터미널에서 본문을 고쳐 재전송합니다. 복사한 명령에는 세션 정보가 포함될 수 있어 공개 저장소에 올리지 않습니다.

세 방법 모두 서버에 보내는 **HTTP 요청 본문을 바꾼다**는 점은 같습니다. URL만 수정하거나 Elements에서 화면의 HTML만 바꾸는 것으로는 이 실습의 요청 본문을 바꾸지 못합니다.

## 관련 CTF·실습 문제

1. **[PortSwigger — Unprotected admin functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)**: 완료(2026-10-15 사용자 확인). 관리자 기능이 공개되어 있는지 확인하는 입문 문제입니다. `/robots.txt` 단서와 관리자 경로에 관해 질문하며 탐색했습니다. **왜 `/robots.txt`를 봤나?** 사이트가 검색 엔진에 방문하지 말라고 알리는 경로가 적혀 있을 수 있어 관리자 경로를 발견하는 단서가 됩니다. **왜 경로를 발견한 뒤에도 접속 결과를 확인해야 하나?** 경로를 아는 것과 그 기능을 사용할 권한이 있는 것은 별개입니다. 서버가 비관리자에게도 기능을 허용하는지 확인해야 합니다. 당시 요청·응답 화면은 이 메모에 보관하지 않았습니다.
2. **[PortSwigger — User role controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)**: 완료(2026-10-15 사용자 확인). 관리자 여부를 조작 가능한 쿠키로 판단하는 실습입니다. 이번 문제와 똑같은 `roleid` 본문 문제는 아니지만, 클라이언트 값을 신뢰하면 어떻게 권한 상승이 생기는지 비교하기 좋습니다. 구체적인 수행 과정은 이 대화에 남아 있지 않습니다.
3. **[OWASP Juice Shop — Admin Registration](https://help.owasp-juice.shop/part2/improper-input-validation.html#register-as-a-user-with-administrator-privileges)**: 완료(2026-10-15 사용자 확인). 일반 사용자 가입 요청에 예상하지 않은 역할 필드를 넣어 관리자 권한 등록을 시험하는 CTF형 문제입니다. 공식 가이드 기준 난이도 3성입니다. 이번 문제와 가장 비슷한 요청 본문 수정 연습입니다. 구체적인 수행 과정은 이 대화에 남아 있지 않습니다.

**추천 순서:** 1 → 2 → 3. 앞의 둘은 PortSwigger 랩이고, 마지막은 OWASP Juice Shop의 점수판 문제입니다. 풀이를 보기 전에 정상 요청·수정한 값·서버 반응을 세 줄로 먼저 적어봅니다.

## 원인과 대응

서버는 이메일 변경 기능에서 허용된 `email`만 처리하고, `roleid` 같은 권한 필드는 일반 사용자의 입력으로 변경되지 않도록 해야 합니다. 관리자 기능에 접근할 때도 현재 서버가 확인한 역할을 기준으로 인가 검사를 해야 합니다. 클라이언트가 보낸 권한 값을 그대로 믿어서는 안 됩니다.
