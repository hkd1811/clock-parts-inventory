# 시계 부속 재고관리 앱 (Wood Clock Workshop)

이 저장소는 `index.html` 하나짜리 재고관리 앱이다. 다른 PC/세션에서 처음 작업을 시작할 때 반드시 읽을 것 — 여기 없는 세부 이력은 git log/commit 메시지에 있다(버전마다 커밋 남기는 관례, 아래 참고).

## 구조

- **`index.html`**: React 18 UMD + Babel standalone(`runtime: "classic"`) + Tailwind CDN. 빌드 스텝 없음, Node/Python 설치 안 되어 있는 환경 기준. 브라우저에서 바로 Babel 트랜스파일.
- **데이터**: 브라우저 `localStorage`에 저장(`cp_ledger_` 접두사 키들: movements/hands/transactions/infoItems/deletedSeedKeys/github_settings).
- **탭 구조**: AI등록 / A타입(태양축) / B타입(정공축) / 무브먼트 / 기타 / 입출고 이력 / Info / 가공계산기. **주의**: "무브먼트"와 "기타" 탭은 같은 `movements` 배열을 `category` 필드로만 나눠서 보여주는 것 — 전체 개수를 셀 때 한쪽 탭만 보고 판단하면 착각하기 쉽다(실제로 이 착각 때문에 v92~v93 사이 삽질한 적 있음).
- **`app.ico`, `shaft-guide.jpg`, `images/`**: 정적 자산. 저장소 루트에 있는 다수의 `*.jpg`는 예전부터 이미지 호스팅용으로 쓰던 것들(제품 사진, `raw.githubusercontent.com` 링크로 참조됨).

## 버저닝 & 배포

- **`APP_VERSION`** 상수(파일 상단 근처)를 사람이 읽을 수 있는 문자열로 매 변경마다 올린다. 배지로 화면에 표시됨(브라우저가 캐시된 옛 화면 보여주고 있는지 확인용).
- **Git 커밋 관례**: `APP_VERSION`을 올릴 때마다 그 버전의 실제 코드 변경사항을 커밋으로 남긴다(과거엔 안 그랬다가 2026-08-03부터 시작). 커밋 메시지에 "무엇을 왜 고쳤는지" 충분히 설명 — 나중에 되돌리거나 원인 추적할 때 유일한 근거가 됨.
- **원격**: `origin` = `https://github.com/hkd1811/clock-parts-inventory.git`, 브랜치는 `main`(로컬은 관례상 `master`로 작업 후 `git push origin master:main`). **공개(public) 저장소** — 이미지도 여기 루트에 같이 있어서 함부로 비공개 전환하면 기존 이미지 링크 다 깨짐.
- **GitHub Pages**로 배포됨: **https://hkd1811.github.io/clock-parts-inventory/** — `main` 브랜치에 push하면 자동 반영(보통 1~2분).
- **주의**: 이 앱은 데이터도 GitHub에 커밋으로 쌓인다(아래 참고) — 그래서 `git push` 전에 거의 항상 `git fetch && git merge`가 필요하다(원격에 앱이 직접 만든 "재고 데이터 동기화 - ..." 커밋들이 코드 push와 무관하게 계속 쌓이기 때문). Fast-forward 실패해도 당황하지 말고 fetch/merge 후 재시도.

## 다기기 데이터 동기화 (2026-08-03, v90~v93에 추가)

"한 번에 한 기기만 사용"을 전제로 한 단순 모델. 관련 함수는 `index.html` 안에서 검색: `githubFetchLedger`, `githubPushLedger`, `syncWithGithub`, `pushLedgerToGithub`, `forceUploadToGithub`.

- `movements`/`hands`/`transactions`/`infoItems`/`deletedSeedKeys` 5개 키를 저장소의 **`data/ledger.json`** 파일 하나로 동기화한다.
- 기존 이미지 업로드용 GitHub 연결 설정(owner/repo/token, `GithubSettingsContext`)을 그대로 재사용 — 토큰은 절대 ledger.json에 포함 안 시킴(동기화 대상 5개 키에 github_settings는 없음, 이 경계 유지할 것).
- **모든 CRUD는 `persist(key, val)` 하나를 거친다** — 새 필드/기능 추가 시 이 함수만 건드리면 동기화까지 자동으로 딸려온다. 개별 CRUD 핸들러를 일일이 고칠 필요 없음.
- **GitHub Contents API는 파일이 1MB 넘으면 `content`를 안 준다**(`encoding:"none"`) — `githubFetchLedger`는 이 경우 `download_url`에서 원문을 직접 받아오도록 폴백되어 있음. 이 폴백 지우지 말 것 — ledger.json이 이미지 base64 때문에 이미 1.5MB 정도 됨.
- **"이 기기 데이터로 GitHub 강제 덮어쓰기" 버튼**(설정 모달 하단, 빨간 글씨): 원격 데이터를 신뢰하지 않고 지금 이 기기 데이터로 무조건 덮어씀. 다른 기기가 먼저 잘못된/오래된 데이터를 올려버렸을 때 복구용. 일반 "저장" 버튼은 반대로 원격이 있으면 그걸 우선 pull하니, 복구 상황에서 실수로 "저장"을 누르면 오히려 진짜 데이터가 덮어써질 수 있음 — 주의.

## 테스트 워크플로우

- Node/Python 없음 → PowerShell로 간단 정적 파일 서버(`New-Object System.Net.HttpListener` 기반 스크립트)를 detached process로 띄우고 `Claude_Browser` MCP 도구(preview_start/navigate/read_console_messages/read_page/javascript_tool)로 검증.
- **`ResizeObserver`/`requestAnimationFrame`이 이 자동화 브라우저 환경에서 전혀 발화하지 않음** — 레이아웃 재측정 등에 의존하지 말고 `setTimeout` 기반 지연 재측정 패턴을 쓸 것(이미 `useLayoutEffect` 부분에 적용되어 있음).
- 합성 `dispatchEvent(new Event('blur'))`는 React의 `onBlur`를 안정적으로 트리거 못함 — 실제 `.blur()` 메서드를 쓸 것.
- 같은 도구 호출 안에서 상태 변경 직후 바로 읽으면 stale한 React 상태를 볼 수 있음 — 별도 호출로 다시 읽을 것.
- 테스트 후엔 `localStorage.clear()`로 정리하고, 띄운 PowerShell 프로세스도 꼭 종료할 것(포트 계속 점유됨).

## 일반 관례

- 신규 입력 필드에서 여러 자리 숫자를 타이핑해야 하면, 매 keystroke마다 clamp하지 말고 raw 문자열 버퍼 + blur/Enter 커밋 패턴을 쓸 것(`CncSpinField` 참고) — 즉시 clamp하면 다중 자릿수 입력이 막힘.
- 브라우저 기본 `<input type="number">` 스핀 화살표는 커스텀 스핀 버튼과 중복/혼란을 주므로 `type="text"` + `inputMode="decimal"`을 씀.
- 삭제(휴지통)는 `window.confirm`, 되돌릴 수 없는 동작(강제 덮어쓰기 등)도 `window.confirm` 필수.
- UI 텍스트는 전부 한국어. 주석은 "왜"를 설명할 때만(숨은 제약, 특정 버그의 우회, 놀라운 동작) — 코드가 이미 보여주는 "무엇"은 쓰지 않음.
