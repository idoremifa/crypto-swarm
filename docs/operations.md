# Operations — 운영 정책

이 문서는 **돌아가고 있는 시스템을 운영하는 사람**을 위한 것이다. 리스크 한도, 장애 대응, 승인 절차, 알림 등급 같은 실전 규칙을 담는다. 코드 구조는 [architecture.md](./architecture.md)에, 권한 경계는 [AGENTS.md](../AGENTS.md)에 있다.

## 리스크 룰

Risk Gate가 적용하는 하드 룰이다. 구체 숫자는 환경별 설정에서 관리한다.
실제 값이 들어간 파일은 Git에 커밋하지 않고, 저장소에는 예시 파일만 둔다.
변경은 [Approval Gate](#승인-절차-approval-gate)를 거친다.
리스크 config의 초기 키 구조 후보는 이 문서의 [Config 파일 포맷](#config-파일-포맷)을 기준으로 한다.

### 범주 (한 줄 요약)

아래 목록은 무엇을 건다는 감각만 준다.
실제 키 이름과 묶음은 Risk Gate 구현 시점에 조정할 수 있다.

- **계좌 레벨**: 일일/주간 손실, 최대 낙폭, 현금 비중 하한, 분당·일일 주문 수, 미체결 주문 수.
- **심볼 레벨**: 보유 비중, 주문당 금액, 유동성/스프레드/슬리피지 한도.
- **전략/에이전트 레벨**: 자본 배정, 일일 손실, 주문 빈도, 연속 손실 시 자동 pause, 오류 시 자동 quarantine.
- **시스템 레벨**: 거래소 장애·market data 지연·DB 쓰기 실패·reconciliation mismatch 시 주문 중지, 배포 직후 warm-up.

한도 상향처럼 권한 경계가 걸린 변경은 [AGENTS.md §1.3 자동 변경](../AGENTS.md#13-자동-변경)을 따른다.

### Risk Gate 출력

- `APPROVED` — 원안대로 진행
- `REDUCED` — 한도 내로 크기 축소 후 진행
- `REJECTED` — 주문 금지
- `PAUSED` — 조건 해소될 때까지 해당 경로 정지

### 주문 안전 원칙

- 거래소 주문 API 호출 전에 `ProposedTrade`, `RiskDecision`, `OrderRequest`, `client_order_id`를 DB에 먼저 저장한다.
- 이 저장이 실패하면 주문을 제출하지 않는다.
- live-small 주문의 `client_order_id`는 업비트 `identifier`로 전송한다.
- 주문 제출 후 응답이 불명확하면 기존 `identifier`로 조회한다. 같은 의도에 새 `identifier`를 발급해 재주문하지 않는다.
- 매도는 보유 현물 수량을 줄이는 exit로만 허용한다. 사용 가능 보유 수량과 미체결 매도를 반영했을 때 순숏이 될 수 있으면 Risk Gate가 `REJECTED`로 기록한다.
- 주문 현재 상태와 모든 상태 전이는 분리해 기록한다. 상태 전이는 append-only로 남긴다.

---

## Config 파일 포맷

리스크 룰의 구체 숫자는 로컬 또는 운영 환경의 설정에 담는다.
MVP에서는 `configs/{dev,paper,live-small}.yaml`을 우선 검토한다.
이 섹션은 리스크 config 키 구조의 초기 후보를 보여준다.
아래 YAML은 키 구조를 설명하기 위한 예시다.
`null` 값은 비워둔 자리다.
실제 운영 값은 [mvp.md D2](./mvp.md#d2-리스크-한도-숫자로)에서 확정한 뒤 환경별 설정에 작성한다.

```yaml
# configs/live-small.example.yaml
mode: live-small                 # dev | paper | live-small

risk:
  account:
    daily_loss_krw: null                 # 일일 최대 손실(KRW)
    weekly_loss_krw: null                # 주간 최대 손실(KRW)
    max_drawdown_pct: null               # 최대 낙폭(%)
    cash_min_ratio_pct: null             # 전체 현금 비중 하한(%)
    orders_per_minute: null              # 분당 최대 주문 수
    orders_per_day: null                 # 하루 최대 주문 수
    open_orders_max: null                # 전체 미체결 주문 수 제한

  symbol:
    max_position_ratio_pct: null         # 심볼별 최대 보유 비중(%)
    max_notional_krw: null               # 심볼별 주문당 최대 금액
    spread_bps_max: null                 # 스프레드 상한(bp)
    slippage_bps_max: null               # 슬리피지 허용 한도(bp)
    min_liquidity_krw: null              # 심볼별 최소 유동성 조건

  agent:
    max_allocation_pct: null             # 에이전트별 최대 자본 배정(%)
    daily_loss_krw: null                 # 에이전트별 일일 손실 한도
    orders_per_minute: null              # 에이전트별 주문 빈도 제한
    pause_on_consecutive_losses: null    # 연속 손실 시 pause 기준(회)

kill_switch:
  daily_loss_ratio: null                 # 일일 손실 한도 대비 도달 비율
  market_data_gap_seconds: null          # market data 단절 허용 시간(초)
  reconciliation_mismatch_seconds: null  # 잔고 불일치 허용 시간(초)
  order_failure_rate_threshold: null     # 주문 실패율 임계치(0~1)
  order_failure_window_minutes: null     # 실패율을 계산할 시간 창(분)

warmup:
  post_deploy_seconds: null              # 배포 직후 주문 금지 시간(초)
```

**규칙**
- 실행하려는 모드의 config에 `null`이 남아 있으면 시작을 실패시킨다.
- 운영자는 자금 규모, 위험 허용도, 심볼 유동성을 보고 직접 값을 정한다.
- [mvp.md D2](./mvp.md#d2-리스크-한도-숫자로)가 확정되기 전에는 `paper`, `live-small` 설정을 실운영에 쓰지 않는다.
- `risk.symbol`과 `risk.agent`는 MVP의 기본 한도 후보다. 심볼·에이전트별 예외가 필요하면 Approval Gate를 거쳐 별도 키를 추가한다.
- Phase 4 이후 `paper`와 `live-small` 변경은 [Approval Gate](#승인-절차-approval-gate)를 통해 적용한다. `configs/*.yaml`을 직접 편집·배포만 하는 것은 금지.
- Phase 0~3의 초기 설정은 코드 리뷰나 작업 기록으로 남긴다.
- `dev` 프로파일은 모의 데이터 전용이므로 더 느슨한 값을 허용하되, 주문 발행 경로는 연결하지 않는다.

---

## kill switch

### 발동 조건 (자동)

- 설정된 일일 손실 비율 도달
- 잔고 불일치(reconciliation mismatch)가 지정 시간 이상 해소되지 않음
- 설정된 시간 창에서 주문 실패율이 임계치 초과
- market data가 지정 시간 이상 끊김

### 수동 발동

CLI와 운영 대시보드(post-MVP)에서 발동할 수 있다.
텔레그램 인바운드로 발동/해제하지 않는다. 자세한 내용은 [AGENTS.md §1.2](../AGENTS.md#12-권한과-비밀)를 참고한다.
아래 명령은 초기 CLI 형태 예시다.
실제 명령 이름과 옵션은 구현하면서 조정할 수 있다.

```bash
uv run python -m apps.worker.main killswitch engage --reason "운영자 판단"
uv run python -m apps.worker.main killswitch status
uv run python -m apps.worker.main killswitch release --approval-id appr_20260419_001 --reason "장애 복구 확인"
```

`killswitch release`는 승인된 Approval ID가 없으면 실패해야 한다. 해제 요청만 만들 때는 Approval Gate CLI로 제안을 생성한다.

### 발동 시 동작

1. 신규 주문을 즉시 중지한다.
2. 미체결 주문 취소를 요청한다.
3. CRITICAL 알림을 전송한다.
4. 리스크 이벤트로 기록한다.

### 해제

해제는 Approval Gate 승인이 필요하다. 발동 원인이 해소됐다는 증거(로그, reconciliation 결과)가 첨부돼야 한다.

---

## 장애 대응

장애는 예외가 아니라 정상 경로다. 각 시나리오는 자동으로 처리되고, 필요한 경우 운영자 알림이 간다.

### 거래소 WebSocket 단절

1. 신규 주문을 중지한다.
2. 공개 시세 stream인지 private 주문·자산 stream인지 구분해 기록한다.
3. 재연결을 시도한다. 지수 backoff를 사용한다.
4. REST snapshot 또는 REST 주문·잔고 조회로 상태를 복구한다.
5. 누락 구간을 표시한다.
6. 정상화 후 warm-up 대기 뒤 재개한다.

### REST API rate limit 또는 5xx 오류

1. 요청 종류와 오류 코드를 기록한다.
2. rate limit은 정해진 backoff 뒤 재시도한다.
3. 5xx 오류는 짧은 재시도 뒤 실패를 명시적으로 기록한다.
4. 주문·잔고 관련 요청이 계속 실패하면 신규 주문을 중지한다.
5. 복구 뒤 reconciliation을 실행한다.

### 중복 이벤트, 누락 가능성

1. 이벤트 ID, sequence(있을 때), payload hash로 중복 여부를 확인한다.
2. 중복 이벤트는 재처리하지 않고 무시 기록만 남긴다.
3. 누락 가능성은 gap으로 표시하고 REST snapshot으로 복구한다.
4. 복구가 불확실하면 신규 주문을 중지하고 ERROR 알림을 보낸다.

### 주문 제출 후 응답 불명

1. 새 `client_order_id`를 만들지 않는다.
2. 기존 `client_order_id`에 매핑된 업비트 `identifier`로 개별 주문을 조회한다.
3. 필요하면 open/closed order 조회로 교차 확인한다.
4. 내부 주문 상태와 주문 이벤트 이력을 보정한다.
5. 불확실하면 신규 주문을 중지하고 CRITICAL 알림을 보낸다.

### 부분 체결, 체결 지연

1. 체결된 수량과 남은 수량을 분리해 기록한다.
2. 남은 주문은 `Open` 또는 `PartiallyFilled` 상태로 유지한다.
3. 지정 시간이 지나면 남은 주문을 취소한다.
4. 체결별 실제 수수료를 수수료 이벤트로 저장한다.
5. 상태가 불확실하면 신규 주문을 중지한 뒤 reconciliation을 실행한다.

MVP에서는 자동 재주문을 기본 동작으로 두지 않는다. 필요하면 별도 정책과 테스트를 추가한다.

### 잔고 불일치

1. 신규 주문을 중지한다.
2. balance / fill / order를 전체 재조회한다.
3. position을 재계산한다.
4. 차이가 허용 오차를 초과하면 운영자 알림을 보낸다.

### 급변동

1. 스프레드/변동성 조건을 확인한다.
2. 신규 진입을 중지한다.
3. 기존 포지션 축소·매도는 Risk Gate가 허용한 명시적 exit 신호가 있을 때만 진행한다.
4. 정상화를 기다린다.

### DB 쓰기 실패

1. 신규 주문을 중지한다.
2. 주문 API 호출 전 저장 실패라면 주문을 제출하지 않는다.
3. 주문 API 호출 후 상태 이벤트 저장 실패라면 즉시 신규 주문을 중지한다.
4. 가능한 경우 로컬 outbox(나중에 재처리할 임시 큐)나 이벤트 로그에 이벤트를 보존한다.
5. ERROR 알림을 보낸다.
6. DB 복구 후 reconciliation을 실행한다.

### 텔레그램 전송 실패

1. 실패한 알림을 알림 이벤트로 기록한다.
2. 지수 backoff로 재시도한다. 최대 횟수를 둔다.
3. CRITICAL 알림 실패가 계속되면 CLI 상태와 로그에 크게 표시한다.
4. 알림 실패만으로 주문 상태를 임의로 바꾸지 않는다.

### 워크플로우 실패

1. 실패한 단계와 입력/출력 요약을 workflow trace에 기록한다.
2. 재시도 가능한 오류는 정해진 횟수·backoff로 재시도한다.
3. 권한 오류·스키마 오류·Risk Gate 거절은 재시도하지 않는다.
4. 주문 관련 workflow가 불확실한 상태가 되면 신규 주문을 중지한다.
5. ERROR 또는 CRITICAL 알림을 보낸다.

---

## 승인 절차 (Approval Gate)

시스템 상태를 바꾸는 제안은 **Approval Gate**를 통과해야 적용된다. MVP는 CLI만 지원한다.

### 승인 대상

- 수수료율 변경 (수수료율 저장소 갱신)
- 에이전트 가중치 변경 (MVP는 동일 가중치 고정, 가중치 기능을 도입한 뒤)
- 전략 파라미터 변경 (파라미터 외부 설정 기능을 도입한 뒤)
- 에이전트 pause/resume
- live-small 한도 변경
- kill switch 해제

### 절차

```
제안 생성 (제안 본문 저장)
  → 승인 이벤트: REQUESTED
  → Telegram 알림 (제안 ID 포함)
  → CLI에서 상세 확인
  → APPROVED 또는 REJECTED
  → 승인 결정 기록 (append-only)
  → 승인된 경우에만 적용 workflow 실행
  → 승인 이벤트: APPLIED 또는 APPLY_FAILED
```

### CLI 사용 예시

아래 명령은 초기 CLI 형태 예시다.
실제 명령 이름과 옵션은 구현하면서 조정할 수 있다.

```bash
uv run python -m apps.worker.main approval list
uv run python -m apps.worker.main approval show appr_20260419_001
uv run python -m apps.worker.main approval approve appr_20260419_001 --reason "공식 페이지 확인"
uv run python -m apps.worker.main approval reject appr_20260419_001 --reason "자동 판독 불확실"
```

Approval ID 포맷은 [architecture.md §ID 생성 규칙 후보](./architecture.md#approval_id)의 초기 후보를 따른다.

### 원칙

- 승인 기록은 **append-only**. 수정·삭제 금지.
- 승인 제안 본문은 생성 뒤 수정하지 않는다. 변경이 필요하면 새 제안을 만든다.
- 승인자와 적용자를 분리할 수 있게 설계한다.
- D6에서 2인 승인이 필요하다고 정한 변경은 승인자 2명을 기록한다.
- 승인 후 적용 직전에도 권한과 스키마를 다시 검증한다.
- 텔레그램 인바운드 조작 금지 원칙은 [AGENTS.md §1.2](../AGENTS.md#12-권한과-비밀)를 참고한다.

### 수수료율 변경 절차

MVP에서는 수수료율 변경 모니터링을 자동화하지 않는다. 다음 절차를 수동으로 운영한다.

1. 운영자가 월 1회 (또는 이벤트 소식이 있을 때) [업비트 수수료 안내 페이지](https://support.upbit.com/hc/ko/articles/900006143046)를 확인한다.
2. 변경이 있으면 수수료율 변경 제안을 CLI로 생성한다.
3. Approval Gate에서 승인한다.
4. 적용 시 기존 수수료율 기록은 적용 종료 시각만 닫는다. 수수료율, 출처, 검증 시각은 덮어쓰지 않는다.
5. 새 수수료율 기록을 추가한다. Approval ID 또는 관련 승인 이벤트 참조를 남긴다.
6. 적용 후 텔레그램 INFO 알림이 간다.

---

## 텔레그램 알림

알림은 발송 전용이다. 텔레그램으로 시스템을 조작할 수 없다.

### 알림 등급

| 등급 | 예시 | 빈도 억제 |
|---|---|---|
| **INFO** | 일일 리포트, paper 요약, 실험 완료, 수수료율 변경 적용 | 묶어 보냄 |
| **WARN** | WebSocket 재연결 반복, 주문 지연, 손실 한도 접근 | 같은 종류는 짧은 시간 내 묶음 |
| **ERROR** | 주문 실패율 급증, 잔고 불일치, DB 쓰기 실패 | 동일 원인 억제 |
| **CRITICAL** | kill switch 발동, 설정된 일일 손실 비율 도달, 실거래 주문 중지 | 억제보다 전달 우선 |

### 초기 알림 대상

- 봇 시작/종료
- 실행 모드 변경 (paper → live-small)
- 업비트 WebSocket 단절/복구
- 업비트 API 오류 증가
- Risk Gate 주문 거절
- 주문 생성/부분 체결/완전 체결/취소
- 일일 손실 한도 접근 또는 설정된 kill switch 비율 도달
- kill switch 발동
- reconciliation mismatch
- 일일 운영 리포트
- Approval 요청 (승인 필요 알림)

### 보안

비밀 값 저장·출력 금지 원칙은 [AGENTS.md §1.2](../AGENTS.md#12-권한과-비밀)을 따른다. 이 섹션은 텔레그램 운영 방법만 정한다.

- `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`는 secret manager 또는 운영 환경 변수로 주입한다.
- `.env`는 로컬 개발 전용이다. 운영/live-small secret은 secret manager 또는 운영 환경 변수로 주입한다.
- 메시지는 원인 파악에 필요한 요약과 짧은 내부 ID만 담는다.

### 전송 정책

- 같은 종류 WARN은 짧은 시간 안에 반복 전송하지 않는다.
- CRITICAL은 중복 억제보다 빠른 전달을 우선한다.
- 전송 실패 시 지수 backoff로 재시도한다.
- 텔레그램 장애가 거래 시스템을 멈추지 않는다.

### 메시지 예

```text
[CRITICAL] kill switch 발동
모드: live-small
원인: 설정된 일일 손실 비율 도달
조치: 신규 주문 중지, 미체결 주문 취소 요청
시간: 2026-04-19 14:32:10 KST
```

```text
[WARN] 업비트 수수료율 확인 필요
마켓: KRW
내부 설정: 0.05%
확인 주기: 월 1회
조치: 공식 페이지 확인 후 수수료율 승인 요청
```

---

## 일상 운영 체크리스트

각 항목은 "무엇을 본다 → 이상 시 누가 무엇을 한다"까지 읽어야 한다. 혼자 운영하는 경우에도 "운영자 본인"이라는 주체를 명시적으로 기록한다.

### 매일

- [ ] 일일 리포트 확인 (수수료 차감 후 순손익)
  → **이상 시 (손실 한도 접근 또는 이해 불가 PnL)**: 운영자가 직접 리스크 이벤트와 당일 체결 원본을 확인한다. 필요하면 신규 주문을 수동 pause한다.
- [ ] CRITICAL/ERROR 알림이 있었는지 확인
  → **이상 시**: 해당 시간대 리스크 이벤트와 로그 원본을 조회한다. 재발 방지 조치가 필요하면 Approval 제안을 생성한다.
- [ ] reconciliation 상태 정상인지 확인
  → **이상 시**: kill switch 자동 발동 여부를 확인한다. 해소되지 않았으면 수동 개입(`killswitch engage`) 후 거래소 잔고와 DB 동기화부터 진행한다.
- [ ] Approval 대기 요청이 있는지 확인
  → **이상 시 (만료 임박 또는 중복 제안)**: 중복은 Reject하고, 만료 임박은 즉시 검토·결재한다.

### 매주

- [ ] 주간 손익, 최대 낙폭 검토
  → **이상 시 (한도 근접)**: 다음 주 한도를 더 타이트하게 조정하는 Approval 제안을 생성한다.
- [ ] 에이전트별 신호 개수와 적중률 검토
  → **이상 시 (적중률 급락 또는 신호 0건)**: 해당 에이전트만 `pause`한다. 자동 quarantine 동작 여부도 함께 확인한다.
- [ ] 실패한 워크플로우 또는 이상 신호 추이
  → **이상 시**: 실패한 workflow trace와 로그 원본을 조사한다. 코드 이슈면 배포 일정에 반영한다.

### 매월 (또는 이벤트 발생 시)

- [ ] 업비트 공식 수수료 페이지 확인
  → **이상 시 (수수료율 변경)**: [수수료율 변경 절차](#수수료율-변경-절차)를 실행한다.
- [ ] API 키 권한 재확인 (출금 권한 없는지)
  → **이상 시 (출금 권한이 생겼다면)**: 즉시 kill switch를 발동하고, 해당 키를 폐기한 뒤 새 조회 전용 키를 재발급한다.
- [ ] 로그/DB 백업 동작 확인
  → **이상 시**: 백업이 동작하지 않으면 다음 주 배포를 막고 백업부터 복구한다.
- [ ] 현재 적용 수수료율과 실행 중인 리스크 config 값 검증. 수수료 seed·제안 입력은 런타임 기준이 아니다. 자세한 내용은 [architecture.md §Upbit Fee Service](./architecture.md#upbit-fee-service)를 참고한다.
  → **이상 시 (예상 값과 불일치)**: Approval 이력으로 마지막 승인된 값을 확인한 뒤 재배포한다.

### Live-small 전환 전 최종 점검

D1~D6 결정값 자체는 [mvp.md §Phase별 의사결정](./mvp.md#phase별-의사결정-decisions-required)에서 관리한다. 전환 직전에는 **결정 완료**와 **실제 리허설** 두 축만 확인한다.

- [ ] D1~D6 모두 확정 (숫자·선택지에 빈칸 없음)
- [ ] paper trading이 D1에서 정한 최소 기간 이상 무장애 동작
- [ ] 텔레그램 알림 수신 테스트 완료
- [ ] kill switch 수동 발동·해제 리허설 완료

---

## 이 문서를 갱신하는 시점

- 실제로 새 장애를 겪었을 때 대응 절차를 추가·수정한다.
- Risk Gate 룰을 추가하거나 수정했을 때 갱신한다.
- 새 승인 대상이 생겼을 때 갱신한다.
- 텔레그램 알림 등급과 대상이 바뀌었을 때 갱신한다.

MVP 기본 장애 시나리오는 유지한다. 이후 새 항목은 실제 장애나 테스트에서 검증된 경우에만 추가한다.
