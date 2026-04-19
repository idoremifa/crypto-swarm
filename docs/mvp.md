# MVP — 지금 할 일

이 문서는 **현재 구현할 범위**와 **Phase별로 정해야 할 것**을 담는다. 장기 비전은 [vision.md](./vision.md)에, 구조 설명은 [architecture.md](./architecture.md)에, 운영 정책은 [operations.md](./operations.md)에 있다.

이 문서는 구현을 시작하기 위한 기준선이다. 모든 세부를 지금 확정하지 않는다.
테이블 컬럼, 파일명, CLI 옵션, 서비스 경계는 각 Phase에서 실제로 만들고 테스트하면서 조정한다.
단, 주문 권한과 금지 규칙은 [AGENTS.md](../AGENTS.md)를 따른다.

## MVP 범위

### 포함

- 업비트 현물 KRW 마켓
- 초기 심볼 2~3개 (유동성 높은 페어)
- Long/flat 전략 (매수 또는 현금 보유만, 숏 금지)
- 3개 전략 에이전트: Momentum, Mean Reversion, Regime Filter
- Risk Gate + Execution Agent + Reconciliation
- Exchange Gateway (업비트), Market Data Service, Feature Service
- backtest / paper / live-small 세 모드
- Upbit Fee Service (주문 전 예상 수수료, 체결 후 실제 수수료 저장)
- Approval Gate (CLI 기반)
- 텔레그램 알림 (INFO/WARN/ERROR/CRITICAL 4등급, 발송 전용)
- PostgreSQL 저장
- 단일 worker 안의 큐와 DB 이벤트 로그
- Agent Orchestrator (asyncio 기반 명시적 workflow 함수)
- 간단한 기간 분리 검증 (학습 구간과 검증 구간 분리)

### 제외 (post-MVP)

다음은 MVP에서 **만들지 않는다.** 자세한 이유는 [vision.md](./vision.md)의 백로그를 참고한다.

- Learning Loop, Post-Trade Review, Agent Score
- Regime Research / Bull / Bear / Debate 에이전트
- Strategy Lab 고급 검증 기능
- walk-forward 검증 자동화 (기간을 순서대로 밀어가며 반복 검증)
- Fee Rate Monitor (수동 확인으로 대체)
- 운영 대시보드 웹 UI (Approval Gate는 CLI만)
- 다른 거래소 어댑터
- Telegram 원격 제어
- Redis Streams 기반 다중 worker 이벤트 버스
- TimescaleDB 최적화
- 자동 재주문 정책

---

## Phase별 의사결정 (Decisions Required)

아래 항목은 **필요한 Phase 직전에** 확정한다. 모든 결정을 한 번에 끝내지 않는다.
해당 Phase에 들어가기 전에는 그 Phase에 필요한 빈칸만 없앤다.

| 결정 | 늦어도 확정할 시점 |
|---|---|
| D1 거래 대상 | Phase 1 시작 전 |
| D2 리스크 한도 | `paper` 값은 Phase 2 전, `live-small` 값은 Phase 5 전 |
| D3 업비트 계정 | 공개 시세 조회는 Phase 1 전 키 없음 확인, 계좌·주문 조회용 키는 Phase 3 전, 주문용 키는 Phase 5 전 |
| D4 텔레그램 | Phase 4 전 |
| D5 실행 환경 | 개발 환경은 Phase 1 전, 운영 환경은 Phase 5 전 |
| D6 승인 정책 | Phase 4 전 |

### D1. 거래 대상

Phase 1 시작 전에는 초기 심볼과 기준 시간 단위만 먼저 정한다.
백테스트 기간, paper trading 기간, live-small 기간은 해당 Phase 직전에 정한다.

- [ ] 초기 심볼 목록 (예: KRW-BTC, KRW-ETH): __________
- [ ] 기준 시간 단위 (1분, 5분, 1시간): __________
- [ ] 백테스트 기간: __________부터 __________까지
- [ ] paper trading 최소 기간 (예: 14일): __________
- [ ] live-small 최소 운영 기간: __________

### D2. 리스크 한도 (숫자로)

리스크 config 키 구조의 초기 후보는 [operations.md의 Config 파일 포맷](./operations.md#config-파일-포맷)이다.
그곳의 YAML은 키 구조를 보여주는 예시일 뿐 값이 확정된 설정 파일이 아니다.
실제 키 이름과 묶음은 Risk Gate 구현 시점에 조정할 수 있다.
이 문서에서는 각 Phase 전에 어떤 값을 확정해야 하는지만 추적한다.

- [ ] live-small 실제 운영 값 확정 (Git에는 커밋하지 않음)
- [ ] paper 실제 모의 운영 값 확정 (Git에는 커밋하지 않음)
- [ ] dev 프로파일은 모의 데이터 전용이며 주문 경로에 연결되지 않는지 확인
- [ ] 심볼 기본 한도가 초기 심볼 전체에 적용하기 충분한지 확인
- [ ] 에이전트 기본 한도가 MVP 전략 에이전트 전체에 적용하기 충분한지 확인
- [ ] `kill_switch`와 `warmup` 값 확정
- [ ] 확정한 값이 운영자 자금 규모, 위험 허용도, 심볼 유동성에 맞는지 검토

### D3. 업비트 계정

- [ ] 업비트 계정 생성 여부
- [ ] Phase 1 공개 시세 조회는 API 키 없이 시작하는 것으로 확인
- [ ] Phase 3 계좌·주문 조회용 API 키 생성 조건 확정 (조회 권한만, 주문·출금 없음)
- [ ] live-small 전용 API 키 생성 조건 확정 (주문 권한 포함, 출금 없음)
- [ ] 생성한 모든 운영 API 키에 IP allowlist 설정
- [ ] 생성한 모든 운영 API 키에 **출금 권한이 없는지 확인**
- [ ] KRW 마켓 거래 수수료율 재확인 (공식 페이지)
- [ ] 수수료 이벤트 적용 여부 확인

### D4. 텔레그램

- [ ] Telegram Bot 생성 (@BotFather)
- [ ] `TELEGRAM_BOT_TOKEN` 확보
- [ ] 알림 받을 `TELEGRAM_CHAT_ID` 확정
- [ ] 야간 알림/mute 시간 정책: __________
- [ ] CRITICAL 알림을 받을 사람: __________

### D5. 실행 환경

- [ ] 실행 위치 (로컬 Mac / VPS / Cloud VM): __________
- [ ] worker 실행 방식 (MVP 기본값: 단일 worker): __________
- [ ] 시간 기준: DB/event 저장은 UTC, 운영 표시와 날짜성 ID는 KST
- [ ] DB 백업 위치: __________
- [ ] Secret 관리 방식 (개발용 `.env`, 운영/live-small은 secret manager 또는 운영 환경 변수): __________

### D6. 승인 정책

- [ ] paper → live-small 전환 조건: __________
- [ ] Approval Gate 승인자: __________
- [ ] 위험도 높은 변경에 2인 승인 요구할지 (혼자 운영하면 "아니오"): __________
- [ ] kill switch를 누를 수 있는 사람: __________

각 항목의 마감 Phase 전까지 필요한 빈칸을 없앤다. Phase 1 데이터 수집은 D1 최소값과 공개 시세 API 키 불필요 확인만 준비되면 시작할 수 있다.

---

## Phase별 체크리스트

### Phase 0: 기획/설계

**산출물**

- [x] 설계 문서 (Phase 0 기준선)
- [ ] Phase 1 착수에 필요한 D1 최소값과 공개 시세 수집 방식 결정
- [ ] 수수료 seed·제안 입력 방식 정리 (공식 페이지 확인 후)
- [ ] `uv`, `pyproject.toml`, 초기 디렉토리 구조 생성

**완료 기준**

- 어떤 컴포넌트가 주문 권한을 갖는지 [AGENTS.md](../AGENTS.md#3-권한-경계)에 명확히 적혀 있다.
- Phase 1에서 사용할 심볼, 시간 단위, 데이터 수집 방식이 적혀 있다.
- live-small 전환 조건은 Phase 5 전까지 확정해야 한다는 마감이 적혀 있다.
- `uv sync`가 성공한다.

---

### Phase 1: 업비트 데이터 수집

**구현**

- [ ] Upbit Exchange Gateway v1 (공개 시세 조회만, 주문·계좌 조회 없음)
- [ ] WebSocket market data 수집 (ticker, trade, orderbook, candle)
- [ ] REST snapshot 동기화
- [ ] 재연결, 중복 이벤트 제거, 누락 가능성 감지
- [ ] PostgreSQL에 시장 이벤트와 candle 기록 저장
- [ ] 기본 Feature Service (returns, MA, RSI, ATR 등)
- [ ] 기본 애플리케이션 로그 설정 (시작/종료, 재연결, 오류)

**완료 기준**

- 24시간 이상 데이터 수집이 끊김 없이 동작한다.
- 재연결 후 누락 구간이 REST snapshot으로 복구된다.
- candle close 전 값이 feature 계산에 들어가지 않는다 (look-ahead bias 방지).
- Phase 1은 업비트 공개 시세 API만 사용하며 API 키나 secret이 필요 없다.

---

### Phase 2: Backtest / Paper 공통 엔진

**구현**

- [ ] 이벤트 기반 backtest 엔진
- [ ] Paper broker (실제 주문 전송 없음, 가상 체결)
- [ ] 슬리피지 모델
- [ ] Upbit Fee Service (예상 수수료, 실제 수수료 저장)
- [ ] 수수료율과 수수료 이벤트 기록
- [ ] 주문 현재 상태와 append-only 주문 이벤트 기록
- [ ] 주문 상태 전이 후보 (Proposed → Approved → Submitted → Open → Filled / Canceled / Expired)
- [ ] 학습 구간과 검증 구간을 분리하는 간단한 검증 도구
- [ ] 전략 실행 인터페이스 후보
- [ ] 벤치마크 전략 2개 (Buy & Hold, 단순 모멘텀)
- [ ] 실험 결과 기록 (DB 또는 파일 export 형태는 구현하면서 결정)

**완료 기준**

- 동일 전략이 backtest와 paper에서 같은 인터페이스로 동작한다.
- 수수료는 체결금액 기준으로 계산되며 매수/매도 정산이 올바르다.
- 수수료 차감 전/후 성과가 둘 다 저장된다.
- 실험 1건이 실행되고 결과가 재현 가능한 형태로 기록된다.

---

### Phase 3: 전략 에이전트 + Risk Gate + Execution

**구현**

- [ ] Agent Orchestrator v1 (asyncio 워크플로우, trace 저장)
- [ ] Momentum Agent, Mean Reversion Agent, Regime Filter Agent
- [ ] Signal Aggregator (충돌 신호 정리, 만료 처리, MVP 기본 동일 가중치)
- [ ] Portfolio Manager (`PortfolioIntent`를 `ProposedTrade`로 변환)
- [ ] Risk Gate (하드 룰 엔진, APPROVED/REJECTED/REDUCED/PAUSED, 보유 수량 초과 매도 거절)
- [ ] Execution Agent (idempotent 주문, 부분 체결, 취소 처리)
- [ ] Reconciliation Service (잔고·주문·체결 재조회)
- [ ] 계좌·주문 조회용 API 키 연결 (조회 권한만, 주문·출금 없음)
- [ ] 주문·잔고 상태 수집 경로 (REST reconciliation 기본, MyOrder/MyAsset private WebSocket은 사용 시 같은 상태 이벤트로 정규화)
- [ ] kill switch (CLI)
- [ ] 기본 장애 대응 경로 (WebSocket 단절, REST 오류, 응답 불명, 부분 체결, 잔고 불일치)

**완료 기준**

- Risk Gate를 우회하는 주문 경로가 코드에 없다.
- 같은 시장 이벤트가 한 번만 처리된다 (idempotency).
- paper 모드에서 전체 경로(데이터 → 신호 → Risk Gate → Execution → Reconciliation)가 돈다.
- 매도 주문은 보유 수량 이하 exit만 생성되고 순숏이 될 경로가 없다.
- 주문 현재 상태와 append-only 주문 이벤트 이력이 함께 남는다.
- 주문 실패/부분체결/취소 시나리오 테스트가 있다.

---

### Phase 4: 운영 도구 (Approval Gate + 텔레그램 + 일일 리포트)

**구현**

- [ ] Approval Gate (CLI 승인/거절, append-only 감사 기록)
- [ ] Approval 제안과 이벤트 append-only 기록
- [ ] Telegram Notification Service (`sendMessage` 전용)
- [ ] 알림 등급 (INFO/WARN/ERROR/CRITICAL) + 중복 억제 + 재시도
- [ ] 알림 발송 이벤트 기록
- [ ] 일일 운영 리포트 생성
- [ ] 수수료율 수동 점검 절차 (seed·제안 입력 검토 → 승인 → 수수료율 저장소 반영)

**완료 기준**

- 테스트 알림 CLI로 메시지를 수신한다.
- 수수료율 변경은 Approval Gate 승인 없이는 반영되지 않는다.
- 텔레그램 장애가 거래 실행을 막지 않는다.
- 텔레그램으로 승인/거절이 불가능한 것을 테스트로 확인한다.
- kill switch 발동 시 CRITICAL 알림이 간다.

---

### Phase 5: 업비트 live-small 전환

**구현**

- [ ] live-small 전용 API 키 연결 (주문 권한 포함, 출금 없음, 조회 키와 분리)
- [ ] 실거래 주문 경로 활성화
- [ ] 실제 체결 수수료 저장 (거래소 응답 우선)
- [ ] live-small 한도 엄격 적용
- [ ] 일일 성과 리포트
- [ ] 운영 런북 (`operations.md` 참조) 점검

**완료 기준**

- D2의 리스크 한도 아래에서 24시간 이상 안정적으로 동작한다.
- 정해진 손실/장애 조건에서 자동 pause가 동작한다.
- 수동 kill switch가 즉시 동작한다.
- paper trading과 live-small의 성과 괴리가 기록되고 D6에서 정한 기준치 이내다.

---

## MVP 졸업 기준 (Phase 5 통과 후)

다음이 모두 충족되면 MVP는 끝이고, 이후 Phase는 [vision.md](./vision.md)의 백로그에서 선택한다.

- [ ] live-small이 2주 이상 예상 한도 내에서 동작한다.
- [ ] 일일 리포트가 자동 생성되고 수수료 차감 후 순손익이 표시된다.
- [ ] 모든 주문에 대해 신호 → Risk Gate 결정 → 체결 → 수수료가 추적된다.
- [ ] 장애 시나리오(WebSocket 단절, 잔고 불일치) 대응이 리허설 또는 실전에서 검증된다.
- [ ] Approval Gate 기록이 append-only로 남아 있다.
- [ ] 운영자가 CLI로 상태를 확인할 수 있다.

## 이 문서를 갱신하는 시점

- Phase 완료 시 체크박스를 업데이트한다.
- D1~D6 결정이 바뀌면 즉시 반영한다. 이 문서는 "미래의 나"를 위한 기록이다.
- 새 Phase가 추가되면 기존 번호를 유지하고 끝에 추가한다. 기존 번호를 유지해야 문서 연결이 깨지지 않는다.
