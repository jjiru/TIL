전체 리로드(Redirect) 대비 비동기 Fetch/Ajax를 통한 UX 최적화
1. 배경 및 문제 상황 (Context & Problem)
쇼핑몰 상품 상세 페이지에서 사용자가 [장바구니 담기]나 [찜하기] 버튼을 누르는 상황을 가정해 본다.

전통적인 Web MVC 방식(Form 전송 후 Redirect)을 사용하면, 버튼을 누를 때마다 화면 전체가 깜빡이며 새로고침(Reload)된다. 이로 인해 사용자가 보던 스크롤 위치가 최상단으로 복구되거나 화면이 로딩되는 짧은 시간 동안 사용자 경험(UX)이 저해되는 문제가 발생한다.

2. 구조적 고민과 원인 분석 (Architectural Thinking)
화면 전체를 다시 그리지 않고, "사용자가 요청한 특정 데이터(장바구니 추가)만 서버로 보내고 결과만 쏙 받아올 수는 없을까?"라는 고민에서 출발했다.

기존 방식 (Form Submit): 브라우저가 화면 전체의 제어권을 서버에 넘김 ➡️ 서버가 새 HTML을 통째로 반환 ➡️ 화면 깜빡임 발생.

개선 방식 (Asynchronous Ajax): 브라우저는 가만히 있고, 자바스크립트(배달원)가 등 뒤에서 서버와 몰래 데이터만 교환 ➡️ 화면 고정, 필요한 부분만 실시간 업데이트.

3. 해결 방안: 비동기 통신 기술의 선택과 비교
비동기 통신을 구현하기 위해 두 가지 선택지를 비교하고, 현재 아키텍처(Spring MVC + JSP)에 맞는 최적의 도구를 선택했다.

① jQuery $.ajax() (선택)
장점: 내부적으로 직렬화(Serialization)나 데이터 가공을 자동 처리해 주어 코드가 간결하고 직관적임.

이유: 어차피 JSP 화면 제어를 위해 jQuery 라이브러리를 로드한 환경이므로, 통신 도구도 $.ajax()를 활용하는 것이 생산성 측면에서 가성비가 높음.

② Vanilla JS fetch() (대안)
장점: 현대 브라우저 표준 내장 API라 외장 라이브러리가 필요 없음.

단점: 데이터 포장(JSON.stringify)이나 응답 파싱(.then())을 개발자가 날것 그대로 직접 제어해야 해서 코드가 다소 무거워 보일 수 있음.

4. 핵심 코드 구현 및 독해 (Code Implementation)
사용자가 [장바구니] 버튼을 눌렀을 때, 화면 이동 없이 서버에서 영수증(SUCCESS 또는 LOGIN_REQUIRED)을 받아와 브라우저를 제어하는 핵심 로직이다.

JavaScript
$("#addCartBtn").click(function() {
    // 1. 전송할 데이터 수집
    let pid = $("input[name='pid']").val();
    let quantity = $("#cartQuantity").val();
    
    // 2. 비동기(Ajax) 요청 송신
    $.ajax({
        url: "${pageContext.request.contextPath}/user/cart/add", 
        type: "POST",
        data: { pid: pid, quantity: quantity }, // 보따리에 데이터 포장
        
        // 3. 서버 응답에 따른 동적 분기 처리 (UX 제어)
        success: function(response) {       
            let res = response.trim();
            
            if(res === "LOGIN_REQUIRED") {
                alert("로그인이 필요한 서비스입니다.");
                location.href = "/user/member/login"; // 로그인 페이지로 유도
                return; 
            }
            if(res === "SUCCESS") {
                if (confirm("장바구니에 상품이 담겼습니다.\n장바구니로 이동하시겠습니까?")) {
                    location.href = "/user/cart/list"; // 사용자가 원할 때만 이동
                }
            }
        },
        error: function() {
            alert("장바구니 담기 중 오류가 발생했습니다.");
        }
    });
});
5. 깨달은 점 및 성찰 (Retrospective)
UI 리모컨과 데이터 보따리의 분리: 화면에 보이는 수량 창(id="quantity")과 서버로 전송되는 숨겨진 폼(id="cartQuantity")을 분리해 두고, 자바스크립트 keyup/change 이벤트를 통해 실시간으로 데이터를 동기화(Synchronization)해 주는 구조의 정석을 배웠다.

현실적인 개발 역량에 대한 고찰: 0부터 코드를 창작해야 한다는 막막함이 있었으나, 실무 SI/SM 생태계에서는 이미 검증된 레퍼런스 코드를 분석하고(독해력), 이를 환경에 맞게 변형·유지보수하는 능력이 더 본질적임을 깨달았다. 코드 한 줄의 배치 이유와 데이터 흐름을 집요하게 추적하는 지금의 공부 방식을 유지하며 기본기를 다지겠다.
