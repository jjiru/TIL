# 전체 리로드(Redirect) 대비 비동기(Fetch API) 처리를 통한 사용자 경험(UX) 최적화

## 1. 배경 및 문제 인식 (Context)
* **기존 방식의 한계:** 순수 MVC 패턴에서 로그인 여부를 체크할 때, 서버에서 `return "redirect:/user/login";`과 같이 직접 페이지를 이동시키는 방식을 주로 사용했습니다.
* **사용자 경험(UX) 저하 인지:** 쇼핑몰 상세 페이지에서 비로그인 유저가 [장바구니 담기] 버튼을 눌렀을 때, 화면 전체가 새로고침(Full Page Reload)되면서 유저가 기존에 보고 있던 스크롤 위치, 탐색 정보 등이 모두 유실되는 불편함을 인지하고 이를 개선하고자 했습니다.

---

## 2. 해결 방안 및 설계 (Approach)

### 🔄 시스템 아키텍처 및 데이터 흐름 비교

#### A. 기존 동기식 Redirect 방식 (화면 뚝뚝 끊김)
1. 브라우저 ──(장바구니 요청)──> 서버 컨트롤러
2. 서버 컨트롤러 ──(로그인 안됨 확인)──> 브라우저에게 "로그인 페이지 주소"로 리다이렉트 명령
3. 브라우저 ──(전체 화면 하얗게 깜빡임)──> 로그인 화면 로딩 (기존 스크롤/상태 증발)

#### B. 개선된 비동기 Fetch API 방식 (현재 구현 방식)
1. 브라우저 자바스크립트(`fetch`) ──(뒷마당에서 몰래 장바구니 요청)──> 서버 컨트롤러
2. 서버 컨트롤러 (`@ResponseBody`) ──(화면 이동 없이 "LOGIN_REQUIRED" 문자열 신호만 반환)──> `fetch` API 수신
3. 자바스크립트 ──(화면 유지한 채 알림창 띄운 후 부드럽게 로그인 페이지로 리다이렉트)

---

## 3. 핵심 소스 코드 분석 (Code Review)

### 🔹 Backend: Spring Controller (상태 신호 반환)
* 컨트롤러 메서드에 `@ResponseBody`를 선언하여 서버의 직접적인 화면 제어권을 프론트엔드로 이관했습니다.
* 세션 검증 후, 비로그인 상태일 경우 다른 페이지로 강제 이동시키는 대신 `LOGIN_REQUIRED`라는 문자열 신호(State String)만 클라이언트에 명확하게 반환합니다.

```java
@PostMapping("/add")
@ResponseBody 
public String addCart(@RequestParam int pid, @RequestParam int quantity, HttpSession session) {
    Object loginUser = session.getAttribute("loginUser");
    
    // 1. 세션 확인 후 비로그인 유저인 경우 신호만 빽(Back) 처리
    if (loginUser == null) {
        return "LOGIN_REQUIRED"; 
    }
    
    // 2. 로그인 유저인 경우 안전하게 형변환(Casting) 후 서비스 로직 수행
    int uno = ((MemberVo) loginUser).getUno(); 
    cartService.addCart(uno, pid, quantity); 
    
    return "SUCCESS"; 
}

🔹 Frontend: JavaScript Fetch API (제어권 확보 및 조건문 처리)
서버가 응답한 문자열을 비동기로 수신하여, 브라우저가 깜빡이지 않는 상태에서 if 조건문을 통해 다음 행동을 정밀하게 제어합니다.

JavaScript
fetch('/add', { method: 'POST', body: formData })
    .then(response => response.text()) 
    .then(data => {
        // 서버의 문자열 신호를 분석하여 프론트엔드가 주도적으로 화면을 제어
        if (data === "LOGIN_REQUIRED") {
            alert("로그인이 필요한 서비스입니다.");
            location.href = "/user/login"; // 기존 브라우저 컨텍스트를 유지하며 부드럽게 이동
        } else if (data === "SUCCESS") {
            alert("장바구니에 상품이 성공적으로 담겼습니다.");
        }
    });
