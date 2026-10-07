# CLAUDE.md

이 파일은 이 저장소에서 작업하는 Claude Code(claude.ai/code)에게 주는 안내서입니다.

## 프로젝트

개인용 할 일 목록 웹앱. 전체가 `index.html` 파일 하나(HTML + 인라인 `<style>` + 인라인 `<script>`)이며, 빌드 도구, 패키지 매니저, 린터, 테스트 프레임워크는 없다. 단일 파일 구조는 요구사항이므로 CSS/JS를 별도 파일로 분리하거나 외부 이미지를 추가하지 않는다. 데이터는 Supabase(`to-do list` 프로젝트, 주소 `https://thkjhebfrnyexheqrdtq.supabase.co`)에 저장하고, `supabase-js`는 CDN `<script>`로 불러온다. 배포는 GitHub `master` 푸시 시 Vercel이 자동으로 한다 (푸시 = 운영 배포이므로 시험 후에 푸시한다). 요구사항은 `PRD.md`, 진행 단계(뼈대 → 기능 → 디자인 → 점검)는 `STEPS.md`에 있다. 문서와 UI 문구는 한국어다.

## 실행과 확인

- 실행: 폴더에서 `python -m http.server 8765` 후 `http://localhost:8765/index.html`로 확인한다 (자동화 도구/미리보기 창은 `file://`을 `data:` URL로 열어 저장소가 막힌다). 끝나면 서버를 종료한다. 운영 DB를 그대로 쓰므로 시험 데이터를 남기지 않는다.
- 자동 테스트는 없다. 검증 기준은 `PRD.md` 6장의 시나리오 12개이며, 브라우저에서 직접 확인한다.
- 시험 중 생긴 데이터는 지운다: 브라우저는 `localStorage.clear()`(익명 세션 초기화), DB는 MCP `execute_sql`로 `delete from auth.users where is_anonymous;` (`todos`는 cascade로 같이 지워진다. 실사용자가 생긴 뒤에는 이 쿼리를 쓰면 안 된다).
- 커밋 메시지는 한국어로 쓰고, 단계별로 커밋해 온 흐름(`뼈대`, `기능`, `디자인`, `점검` 등)을 따른다.

## 구조 (`index.html` 내부)

- **상태**: 메모리의 `todos` 배열 하나 (`{id, text, done}`)가 화면의 기준이고, Supabase `todos` 테이블이 원본이다. 날짜 기반 초기화는 하지 않는다.
- **시작**: `init()`이 세션이 없으면 `signInAnonymously()`로 익명 계정을 만들고 목록을 읽는다. 끝나기 전에는 입력창/추가 버튼을 비활성화하고 빈 상태 문구도 숨긴다 (`loaded` 플래그).
- **데이터 흐름**: 체크/삭제는 `todos`를 바꾸고 → `render()` → `sync(요청)` 순서(낙관적 업데이트)이며, 요청이 실패하면 `reload()`로 서버 상태에 맞춘다. 추가는 서버 insert 성공 후 `todos`에 넣고 `render()`한다. `render()`는 목록을 매번 통째로 다시 그리고, 남은 개수(`done`이 아닌 항목 수)와 빈 상태 문구도 여기서 갱신한다. 개수를 따로 관리하지 않는다.
- **DB**: `todos(id uuid, user_id default auth.uid(), text 1~100자 check, done, created_at)`. RLS는 `authenticated`(익명 포함)가 `user_id = auth.uid()`인 행만 접근하게 한다. 이 프로젝트는 테이블 권한을 자동 부여하지 않으므로 `authenticated`에 select/insert/update/delete만 명시적으로 `grant`했다. 스키마를 바꾸면 MCP `apply_migration`으로 하고, 새 테이블에도 RLS와 `grant`를 함께 설정한다.
- **이벤트**: 추가는 `<form>`의 submit으로 처리해 버튼 클릭과 Enter를 한 경로로 합쳤다. 체크/삭제는 `#todo-list`에 단일 클릭 핸들러(이벤트 위임)로 처리한다. 체크박스와 항목 글자 클릭 둘 다 토글된다.

## 지켜야 할 점

- 항목 텍스트는 반드시 `textContent`로 넣는다 (`innerHTML` 금지, XSS 방지).
- `render()`가 DOM을 다시 만들기 때문에 포커스가 사라진다. 클릭 핸들러 끝에서 같은 위치의 체크박스(없으면 입력창)로 포커스를 복원하는 코드를 유지해야 키보드 조작이 된다.
- 한글 IME 조합 중 Enter로 중복 추가되지 않도록 `compositionstart/end`, `isComposing`, `keyCode 229` 처리가 들어 있다. submit/keydown 핸들러를 고칠 때 깨지지 않게 주의한다.
- 서버 연결/저장이 실패해도 앱이 멈추면 안 된다. `showStatus()`로 안내 문구를 보여준다.
- publishable 키(`sb_publishable_…`)는 브라우저에 공개되도록 만들어진 키라 코드에 있어도 된다. 접근 제어는 전적으로 RLS에 달려 있으므로 RLS를 끄지 않는다. `service_role`/secret 키는 절대 넣지 않는다.
- 입력 길이 제한은 `maxlength="100"`(화면)과 DB check 제약(서버) 두 곳이다.
- 디자인: 노을 배경은 SVG data URI로 CSS 안에 직접 그렸고, 글꼴(`Black Han Sans`, `Jua`)만 Google Fonts에서 불러온다 (오프라인이면 시스템 한글 글꼴로 대체). 터치 대상은 44px 이상, 입력창 글자는 16px 이상을 유지한다.
- 범위 밖: 로그인 화면, 직접 만든 서버, 마감일, 카테고리, 항목 수정, 필터/정렬.
