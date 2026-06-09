# UI 입력창 - Hidden Form 간 데이터 무결성을 위한 실시간 동기화 및 유효성 검증

## 1. 배경 및 문제 인식 (Context)
* **기존 구조의 한계:** 동일한 상품 상세 페이지 내에 [바로 구매하기]와 [장바구니 담기] 기능이 각각 독립된 `<form>` 구조로 분리되어 있었습니다. 이로 인해 유저가 화면에 노출된 메인 수량 변경 창의 값을 수정하더라도, 숨겨진 폼 내부의 `quantity` 데이터에는 실시간으로 반영되지 않는 데이터 불일치(Data Inconsistency) 리스크가 존재했습니다.
* **데이터 무결성(Data Integrity) 확보 필요성:** 유저가 오입력(음수, 비숫자형 문자열 타이핑 등)을 하거나, 수량을 변경한 후 어떤 버튼을 누르더라도 백엔드로 정확한 수량 데이터가 전송되도록 프론트엔드 단에서의 정밀한 제어와 방어 코드가 필요함을 인지했습니다.

---

## 2. 해결 방안 및 설계 (Approach)

### 🔄 시스템 아키텍처 및 데이터 흐름 비교

#### A. 기존 동기화 부재 방식 (데이터 유실 및 에러 위험)
1. 유저가 화면 수량창에서 값을 `5`로 변경 ➡️ 화면 수량창 변수만 `5`로 변경됨.
2. [장바구니] 또는 [바로구매] 버튼 클릭 ➡️ 숨겨진 폼들은 여전히 초기 설정값인 `1`을 전송함 (유저 의도 반영 실패).
3. 만약 유저가 입력창에 영어 글자(`abc`)를 입력하고 전송할 경우 ➡️ 백엔드 서버에서 형변환(Integer Parsing) 에러 발생 (시스템 불안정).

#### B. 개선된 실시간 이벤트 동기화 방식 (현재 구현 방식)
1. 유저의 마우스 조작(`change`) 및 키보드 타이핑(`keyup`) 제스처 발생 ➡️ 자바스크립트가 즉시 날것의 값을 포착.
2. [Client-side Guard Clause] 1 미만의 음수나 숫자가 아닌 글자(`isNaN`) 검출 시 ➡️ 최소 수량인 `1`로 강제 보정 및 필터링.
3. 안전함이 검증된 무결한 데이터를 화면 새로고침 없이 **독립된 두 개의 숨겨진 폼 필드로 동시에 실시간 복사(`Write`)**.

---

## 3. 핵심 소스 코드 분석 (Code Review)

### 🔹 Backend: HTML Hidden Forms (데이터 전송 보따리)
* 실제 서버로 전송을 담당하는 두 개의 폼은 `style="display:none;"` 처리를 통해 유저 눈에 보이지 않게 숨겨두고, 자바스크립트가 데이터를 안전하게 서빙해 줄 수 있도록 명확한 `id` 식별자를 부여했습니다.

```html
<form id="directOrderForm" action="/user/order/checkout_form" method="post" style="display:none;">
    <input type="hidden" name="pid" value="${product.pid}">
    <input type="hidden" name="quantity" id="hiddenQuantity" value="1">
</form>

<form id="cartForm" style="display:none;">
    <input type="hidden" name="quantity" id="cartQuantity" value="1">
</form>
🔹 Frontend: jQuery Event Handling (동기화 및 제어권 확보)
change와 keyup 이벤트를 멀티 바인딩하여 딜레이 없는 실시간 감시 체계를 구축하고, 조건문을 통해 백엔드로 인입될 데이터를 주도적으로 정제합니다.

JavaScript
// 1. 화살표 클릭(change)과 키보드 타이핑(keyup) 이벤트를 동시에 감시
$("#quantity").on("change keyup", function() {
    
    // 유저가 실제 눈에 보이는 입력창에 입력한 현재 수량 값을 추출
    let currentQty = $(this).val();
    
    // 2. [유효성 검사] 숫자가 아니거나(isNaN) 1보다 작으면 강제로 '1'로 고정 (안전장치)
    if(currentQty < 1 || isNaN(currentQty)) {
        currentQty = 1;
    }
    
    // 3. 검증 완료된 안전한 데이터를 숨겨진 Form 태그들에 실시간 복사
    $("#hiddenQuantity").val(currentQty); // 바로구매 폼의 수량 업데이트
    $("#cartQuantity").val(currentQty);   // 장바구니 폼의 수량 업데이트
});
4. 깨달은 점 및 성찰 (Retrospective)
프론트엔드 유효성 검사의 가치: 서버 단에서도 2차 검증을 수행하겠지만, 클라이언트 사이드에서 1차로 비정상 데이터를 필터링해 줌으로써 잘못된 데이터가 백엔드로 유입되어 시스템 장애(Type Mismatch 등)를 유발하는 것을 원천 차단하는 설계의 중요성을 깨달았습니다.

jQuery 라이브러리의 실무적 효용성: 순수 자바스크립트(Vanilla JS)로 작성했다면 다소 번거로웠을 복수 이벤트 멀티 바인딩(change keyup) 처리를 jQuery의 .on() 문법을 통해 간결하고 직관적으로 해결하여, 왜 많은 레거시/Spring 환경에서 여전히 jQuery가 유지보수용 핵심 도구로 애용되는지 몸소 체감할 수 있었습니다.
