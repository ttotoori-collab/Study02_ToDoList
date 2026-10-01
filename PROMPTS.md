# 단계별 프롬프트

[PRD.md](PRD.md)를 Claude Code에서 그대로 쓸 수 있는 5단계 프롬프트로 옮긴 것이다.
각 단계의 프롬프트 블록을 순서대로 복사해 붙여 넣는다. 한 단계가 끝나야 다음 단계로 넘어간다.

## 사용법

1. 프로젝트 폴더에서 `claude`를 실행한다
2. 아래 **공통 전제**를 첫 메시지로 한 번 붙여 넣는다
3. 단계 1부터 5까지 프롬프트를 순서대로 붙여 넣는다
4. 각 단계의 "완료 기준"을 직접 확인한 뒤 다음 단계로 간다

검증은 전부 브라우저에서 한다. `index.html?test`를 열고 페이지 하단 패널의
마지막 줄 `TESTS: <통과> passed, <실패> failed`를 읽는다.

---

## 공통 전제

모든 단계에 적용된다. 첫 메시지로 한 번만 붙여 넣으면 된다.

```
이 프로젝트의 제약을 먼저 알려준다. 앞으로 모든 작업에 이 제약이 적용된다.

- 구현은 index.html 단일 파일이다. HTML, CSS, JavaScript를 전부 이 파일 안에 인라인한다.
  추가 파일을 만들지 않는다.
- file:// 로 더블클릭해서 열었을 때 동작해야 한다.
  type="module", import, fetch 를 쓰지 않는다.
- 외부 의존성이 0이다. CDN 링크, 웹폰트 로드, package.json, 빌드 단계가 모두 없다.
- localStorage 키는 "todolist.v1", 손상 데이터 백업 키는 "todolist.v1.corrupt" 이다.
- 카테고리는 3개로 고정이다.
  work = 업무 = #2563eb, personal = 개인 = #16a34a, study = 공부 = #9333ea
- 할 일 텍스트는 공백 정규화 후 1자 이상 200자 이하다. 입력란에 maxlength="200" 을 준다.
- 화면 출력은 textContent 만 쓴다. innerHTML 을 쓰지 않는다.
- ID 생성에 crypto.randomUUID() 를 쓰지 않는다.
  Date.now().toString(36) + "_" + Math.random().toString(36).slice(2, 7) 를 쓴다.
- 상태가 바뀌면 목록 전체를 다시 그린다. 바뀐 줄만 골라 갱신하지 않는다.
- 목록 이벤트 리스너는 ul 하나에만 붙인다. 이벤트 위임으로 처리한다.
- 본문 최대 폭은 600px, 가운데 정렬이다.
- 필터 선택은 저장하지 않는다. 새로고침하면 미완료 + 전체 카테고리로 돌아간다.
- 저장 상태 객체는 { version: 1, todos: [...] } 형태를 유지한다.

작업 방식은 테스트 우선이다. 각 단계에서 실패하는 테스트를 먼저 쓰고,
실패를 눈으로 확인한 뒤, 통과시키는 최소한의 코드를 쓴다.
단계가 끝나면 커밋한다.
```

---

## 단계 1 — 골격, 테스트 하네스, 저장소 층

**목표:** 파일을 만들고, 테스트를 돌릴 수단을 먼저 갖추고, localStorage 읽기·쓰기를 완성한다.

```
index.html 을 새로 만들고 두 가지를 구현해라.

(1) 파일 골격과 인페이지 테스트 하네스

HTML5 기본 골격(lang="ko", charset, viewport, title "할 일 관리")에
빈 style 태그, body 안에 <div id="test-panel"></div>, 그리고 script 태그를 둔다.

script 맨 위 CONFIG 구획에 상수를 둔다.
  CATEGORIES = { work: {label:"업무", color:"#2563eb"},
                 personal: {label:"개인", color:"#16a34a"},
                 study: {label:"공부", color:"#9333ea"} }
  STORAGE_KEY = "todolist.v1"
  CORRUPT_KEY = "todolist.v1.corrupt"
  MAX_LEN = 200

테스트 하네스를 script 맨 아래에 만든다.
  test(name, fn)          등록만 한다
  assertEq(actual, expected, msg)    !== 이면 기대값과 실제값을 담아 throw
  assertDeep(actual, expected, msg)  JSON.stringify 로 비교해 다르면 throw
  runTests()              등록된 테스트를 try/catch 로 돌리고, 각 줄을
                          #test-panel 에 textContent 로 "PASS <이름>" 또는
                          "FAIL <이름>: <메시지>" 로 쓴다. 마지막 줄에
                          "TESTS: <n> passed, <m> failed" 를 쓴다.
파일 끝에서 location.search 에 test 가 있을 때만 runTests() 를 부른다.

하네스가 제대로 세는지 확인해야 하니, 통과하는 테스트 하나와
일부러 실패하는 테스트 하나를 먼저 넣고 "TESTS: 1 passed, 1 failed" 를
확인한 다음, 일부러 실패하는 쪽을 지워라.

(2) STORAGE 층

loadState(store = localStorage)
  -> { state: { version: 1, todos: [...] }, error: null | "corrupt" | "unavailable" }
saveState(state, store = localStorage)
  -> { ok: boolean, error: string | null }

store 는 getItem / setItem / removeItem 을 가진 객체다. 테스트에서 가짜 store 를
넘길 수 있어야 한다.

loadState 의 판정 순서는 이렇다.
  getItem 자체가 예외를 던지면 -> error "unavailable"
  값이 null 이면 -> 빈 상태, error null
  JSON 파싱이 실패하거나 parsed.todos 가 배열이 아니면
    -> 원본 문자열을 CORRUPT_KEY 로 저장한 뒤 빈 상태, error "corrupt"
  그 외 -> { version: 1, todos: parsed.todos }

먼저 아래 테스트를 쓰고 실패를 확인한 뒤 구현해라.
  - 빈 저장소는 빈 목록을 돌려준다
  - 저장한 상태를 그대로 다시 읽는다
  - 깨진 JSON 은 빈 목록을 주고 원본을 CORRUPT_KEY 에 백업한다
  - todos 가 배열이 아닌 유효 JSON 도 corrupt 로 처리한다
  - setItem 이 예외를 던지면 ok:false 를 돌려준다
  - getItem 이 예외를 던지면 error 가 unavailable 이다

끝나면 커밋해라.
```

**완료 기준:** `index.html?test` 의 마지막 줄이 `TESTS: 7 passed, 0 failed`

---

## 단계 2 — 할 일 로직

**목표:** DOM을 전혀 모르는 순수 함수로 추가·수정·삭제·완료·일괄정리를 만든다.

```
index.html 의 STORAGE 아래에 LOGIC 구획을 만들어라.
모두 순수 함수이고 DOM 을 참조하지 않는다. 새 배열을 돌려주되,
입력이 거부되는 경우에는 원본 todos 를 그대로(같은 참조로) 돌려준다.

normalizeText(text) -> string
  연속 공백(개행 포함)을 공백 하나로 바꾸고 trim 한다
makeId(now = Date.now()) -> string
addTodo(todos, text, category, now = Date.now(), idFn = makeId) -> Todo[]
toggleTodo(todos, id, now = Date.now()) -> Todo[]
editTodo(todos, id, text) -> Todo[]
removeTodo(todos, id) -> Todo[]
clearCompleted(todos, category = "all") -> Todo[]

Todo = { id, text, category, done, createdAt, completedAt }

addTodo 는 idFn(now) 가 만든 id 가 todos 에 이미 있으면 다시 호출한다(최대 5회).
toggleTodo 는 done 을 뒤집고, 완료로 바뀌면 completedAt 에 now 를, 해제되면
null 을 넣는다.
clearCompleted 는 category 가 "all" 이면 전체에서, 특정 키면 그 카테고리
안에서만 완료 항목을 지운다.

먼저 아래 테스트를 쓰고 전부 실패하는 것을 확인한 뒤 구현해라.
  - addTodo 가 기본값(done false, completedAt null, createdAt now)을 채운다
  - 공백만 입력하면 원본을 그대로 돌려준다
  - "앞\n뒤   사이" 가 "앞 뒤 사이" 로 정규화된다
  - 정규화 후 200자를 넘으면 거부하고, 정확히 200자는 받아들인다
  - idFn 이 이미 있는 id 를 돌려주면 다시 생성한다
  - toggleTodo 를 두 번 하면 done 과 completedAt 이 원래대로 돌아온다
  - editTodo 가 텍스트를 바꾸고, 빈 텍스트는 거부한다
  - removeTodo 가 해당 id 만 지운다
  - clearCompleted 가 "all" 과 특정 카테고리에서 각각 올바르게 동작한다
  - 없는 id 로 toggle / remove 하면 원본을 그대로 돌려준다

테스트에서 쓸 공용 헬퍼를 테스트 구획 맨 위에 둬라.
  const T = (over = {}) => ({ id:"a", text:"x", category:"work",
                              done:false, createdAt:1, completedAt:null, ...over });

끝나면 커밋해라.
```

**완료 기준:** `TESTS: 17 passed, 0 failed`

---

## 단계 3 — 필터·정렬·진행률과 화면 그리기

**목표:** 보여줄 목록과 진행률을 계산하고, 그것을 화면에 그린다.

```
두 가지를 구현해라.

(1) 파생 상태 구획 (LOGIC 아래, 여전히 DOM 을 모르는 순수 함수)

visibleTodos(todos, statusFilter, categoryFilter) -> Todo[]
  statusFilter 는 "active" | "all" | "done", categoryFilter 는 "all" 또는 카테고리 키
  두 필터는 AND 로 결합한다
  정렬: 미완료는 createdAt 오름차순, 완료는 completedAt 내림차순,
       "all" 일 때는 미완료 전부가 먼저 오고 그 뒤에 완료가 온다
  원본 배열을 변형하지 않는다

progress(todos, category = "all") -> { done, total, percent }
  percent 는 0~100 정수(반올림), total 이 0 이면 percent 도 0

(2) 화면 골격과 RENDER 구획

body 에 다음을 둔다.
  #warning (기본 숨김), #progress,
  #add-form (input#text[maxlength=200], select#category, 제출 버튼),
  필터 바 ([data-status] 버튼 3개, select#filter-category, button#clear-done),
  ul#list, #empty, #test-panel
  template#todo-item 안에 li 한 줄의 구조를 둔다:
    input[type=checkbox], .text, .badge,
    button[data-action="edit"], button[data-action="delete"]

categoryOf(key) -> { label, color }
  모르는 키면 { label: "기타", color: "#6b7280" } 를 돌려준다
renderList(listEl, todos, editingId) -> void
  listEl 을 비우고 template 을 복제해 채운다. 완료 항목의 li 에 done 클래스를 준다.
  editingId 와 같은 항목은 .text 대신 input.edit-input 을 넣고 현재 텍스트를 value 로 준다.
renderProgress(progressEl, todos) -> void
  전체 바와 "done/total (percent%)", 그 아래 카테고리 3종의 "라벨 done/total" 을 쓴다
render(root, state, ui) -> void
  ui = { statusFilter, categoryFilter, editingId, warning }
  경고 배너, 진행률, 목록, 필터 버튼 활성 표시, 빈 상태 메시지를 처리한다
  빈 상태 메시지는 두 경우를 구분한다.
    할 일이 0개면 "할 일이 없습니다. 위에서 추가해 보세요."
    필터 결과만 0건이면 "조건에 맞는 할 일이 없습니다."

CSS: 본문 max-width 600px 가운데 정렬, 시스템 폰트, 배지는 카테고리 색을 글자색으로
쓰고 배경은 같은 색의 연한 틴트, .done .text 는 취소선과 흐린 색.

먼저 아래 테스트를 쓰고 실패를 확인한 뒤 구현해라.
  - 정렬이 미완료 추가순 / 완료 최근순으로 나온다
  - visibleTodos 가 원본 배열을 변형하지 않는다
  - 상태 필터와 카테고리 필터가 AND 로 결합한다
  - progress 가 1/3 을 33% 로 돌려준다
  - 할 일이 없으면 0/0 0% 를 돌려준다
  - 카테고리별 진행률을 따로 센다
  - renderList 가 항목 수와 순서를 반영한다
  - 완료 항목에 done 클래스가 붙는다
  - 다시 그리면 이전 항목이 남지 않는다
  - "<img src=x onerror=alert(1)>" 를 넣어도 HTML 로 해석되지 않는다
  - 배지에 카테고리 라벨이 들어간다
  - 모르는 카테고리는 "기타" 로 그린다
  - editingId 와 같은 항목이 input.edit-input 으로 그려진다
  - renderProgress 출력에 "1/2", "50%", "업무" 가 모두 들어간다

renderList 테스트는 document.createElement("ul") 로 만든 분리된 요소에 그려서 확인해라.

끝나면 커밋해라.
```

**완료 기준:** `TESTS: 31 passed, 0 failed`

---

## 단계 4 — 추가·완료·삭제·필터 동작 연결

**목표:** 실제로 쓸 수 있는 앱이 된다. 새로고침해도 데이터가 남는다.

```
index.html 에 EVENTS 구획을 만들어 화면과 로직을 연결해라.

모듈 스코프 상태를 둔다.
  state = { version: 1, todos: [] }
  ui = { statusFilter: "active", categoryFilter: "all", editingId: null, warning: null }

setTodos(nextTodos)
  state.todos 를 교체하고 saveState 를 부른다. 실패하면 ui.warning 을 세운다.
  그리고 전체를 다시 그린다.
setUi(patch)
  ui 에 병합하고 다시 그린다. 저장하지 않는다.
mount(root)
  loadState() 결과를 state 에 넣는다. error 가 "corrupt" 면
  "저장된 데이터가 손상되어 새로 시작합니다", "unavailable" 이면
  "이 브라우저에서는 저장되지 않습니다" 를 ui.warning 에 넣는다.
  리스너를 붙이고 최초 렌더를 한다.

연결할 동작은 넷이다.
  #add-form submit: preventDefault 후 addTodo 로 추가하고 setTodos 한다.
    추가 후 입력란을 비우되 포커스는 그대로 두고, 카테고리 선택은 바꾸지 않는다.
    (하루 10~20개를 연속으로 입력하는 흐름이라 이게 중요하다)
  #list click 하나로 위임: 가장 가까운 li[data-id] 와 target 의 data-action 으로 분기한다.
    체크박스는 toggleTodo, data-action="delete" 는 confirm 을 거친 뒤 removeTodo.
  [data-status] 버튼과 #filter-category 변경: setUi 로 필터를 바꾼다.
  파일 끝에서 ?test 가 아닐 때만 mount(document.body) 를 부른다.

saveState 를 테스트에서 바꿔 끼울 수 있어야 하므로 function 선언 대신
let saveState = function (...) 형태로 둬라. 그리고 아래 테스트를 추가해라.
  - 저장에 실패하면 ui.warning 이 세워지고 state.todos 는 그대로 유지된다
    (테스트가 끝나면 원래 saveState 와 상태를 되돌려 놓는다)

끝나면 브라우저에서 ?test 없이 열고 직접 확인해라.
할 일 3개를 Enter 로 연속 추가하고, 하나를 완료 체크하고, 새로고침한다.
3개가 남아 있고 완료 상태가 유지되며 진행률이 1/3 (33%) 여야 한다.

끝나면 커밋해라.
```

**완료 기준:** `TESTS: 32 passed, 0 failed` 이고, 새로고침 후에도 할 일과 완료 상태가 유지된다

---

## 단계 5 — 인라인 수정, 완료 비우기, 마무리 검증

**목표:** 남은 기능을 채우고 손으로 전부 확인한 뒤 제출 상태로 만든다.

```
마지막 두 기능을 연결하고 전체를 검증해라.

(1) 인라인 수정
  #list 위임 핸들러에 data-action="edit" 분기를 더해 setUi({ editingId: id }) 를 한다.
  input.edit-input 의 keydown 에서
    Enter 면 editTodo 로 반영하고 setTodos 한 뒤 editingId 를 null 로,
    Escape 면 저장하지 않고 editingId 만 null 로 되돌린다.
  다른 항목의 수정 버튼을 누르면 editingId 가 교체되므로 이전 편집은 자동 취소된다.
  모달은 쓰지 않는다.

(2) 완료 비우기
  #clear-done 클릭 시 대상 개수를 progress(state.todos, ui.categoryFilter).done 으로 구한다.
  "완료한 할 일 N개를 삭제합니다. 계속할까요?" 로 confirm 을 띄우고,
  통과하면 clearCompleted(state.todos, ui.categoryFilter) 를 setTodos 한다.
  render 에서 대상이 0이면 버튼에 disabled 를 건다.

(3) 전체 테스트를 다시 돌려 회귀가 없는지 확인해라.

(4) ?test 없이 열고 아래 7가지를 손으로 확인해라. 실패하면 고치고 다시 돈다.
  1. 할 일 5개 추가 후 새로고침하면 5개가 그대로 있다
  2. 완료 체크 후 새로고침하면 완료 상태가 유지된다
  3. 필터 조합(미완료/전체/완료 × 전체/업무)을 바꿔도 목록이 올바르다
  4. 빈 목록과 필터 결과 0건의 메시지가 서로 다르다
  5. 수정 중 Escape 가 원래 값을 되돌린다
  6. 삭제 confirm 에서 취소하면 삭제되지 않는다
  7. 완료 비우기가 미완료 항목은 남기고 완료 항목만 지운다

(5) README.md 의 "현재 상태" 를 설계 단계에서 구현 완료로 바꾸고,
    실행 방법(index.html 더블클릭)과 테스트 방법(index.html?test)을 적어라.

끝나면 커밋하고 푸시해라.
```

**완료 기준:** 테스트가 전부 통과하고, 수동 확인 7개가 전부 통과하고, 푸시가 끝난 상태

---

## 단계 요약

| 단계 | 내용 | 끝났을 때 |
|---|---|---|
| 1 | 골격 · 테스트 하네스 · 저장소 층 | 7 passed |
| 2 | 할 일 로직 (추가/수정/삭제/완료/정리) | 17 passed |
| 3 | 필터 · 정렬 · 진행률 · 렌더링 | 31 passed |
| 4 | 이벤트 연결 — 추가/완료/삭제/필터 | 32 passed, 새로고침 유지 |
| 5 | 인라인 수정 · 완료 비우기 · 전체 검증 | 수동 확인 7개 통과 |

4단계가 끝나면 이미 쓸 수 있는 앱이다. 5단계는 남은 기능과 검증이다.
