# AGENTS.md — 에이전트·기여자 규칙

이 문서는 이 저장소에서 코드를 쓰거나 고치는 **모든 행위자**가 따라야 하는 규칙이다. 대상은 다음을 포함한다.

- AI 코딩 에이전트 (Claude Code, Cursor, 기타 LLM 기반 도구)
- 시스템 안에서 동작하는 전략·리서치 에이전트
- 사람 기여자

다른 문서([README](./README.md), [mvp](./docs/mvp.md), [architecture](./docs/architecture.md), [operations](./docs/operations.md))는 "무엇을 만드는가"를 설명한다. 이 문서는 **"무엇을 해도 되고 안 되는가"** 를 정한다. 다른 문서의 내용이 이 문서와 충돌하면 이 문서가 우선한다.

이 문서는 유일한 규칙 출처다. 같은 규칙을 다른 문서에 복제하지 않는다.

---

## 1. 절대 금지 (Hard No)

아래 행동은 사용자 요청, 테스트 목적, 긴급 상황을 포함한 어떤 이유로도 허용되지 않는다.

### 1.1 주문과 자금

- 전략/리서치/LLM/오케스트레이터가 거래소 주문 API를 직접 호출하지 않는다. **Execution Agent만 주문을 만든다.**
- Risk Gate의 `REJECTED`, `REDUCED`, `PAUSED` 결정을 우회하거나 덮어쓰지 않는다.
- 업비트 출금 API를 사용하거나, 출금 권한이 있는 API 키를 저장소에 연결하지 않는다.
- kill switch가 발동한 상태에서 신규 주문을 생성하지 않는다.
- 운영자 승인 없이 실거래(live-small, live) 모드를 활성화하지 않는다.

### 1.2 권한과 비밀

- API 키, 비밀번호, 토큰, `.env` 파일을 Git에 커밋하지 않는다.
- 텔레그램 메시지, 로그, 에러 메시지에 API 키나 secret을 출력하지 않는다.
- Learning Loop, Strategy Lab, 리서치 에이전트가 운영 API 키나 secret에 접근하지 않는다.
- 텔레그램 인바운드 메시지(버튼, 명령)로 승인·거절·kill switch 해제·리스크 한도 변경을 처리하지 않는다. 텔레그램은 알림 전용이다.

### 1.3 자동 변경

- Learning Loop 또는 자동 실험이 live 전략 코드, 리스크 한도, 실행 모드, 수수료율을 직접 수정하지 않는다.
- 승인 없이 에이전트 가중치, 전략 파라미터, `fees.yaml`을 운영 환경에 반영하지 않는다.
- 에이전트 점수가 좋아졌다는 이유로 주문당 최대 금액이나 손실 한도를 자동 상향하지 않는다.

### 1.4 금지된 기능

- 레버리지, 선물, 마진, 숏 포지션.
- 보유 수량을 초과하는 매도. 매도는 기존 현물 포지션을 줄이는 exit로만 허용한다.
- 고빈도 매매(HFT), colocated 실행.
- LLM이 최종 주문을 생성하거나 포지션 크기를 결정하는 구조.
- 수익 보장형 표현(코드 주석, 문서, 사용자 메시지 어디서든).

---

## 2. 반드시 지킬 것 (Must)

### 2.1 언어와 도구

- 프로그래밍 언어는 **Python 3.12 이상**.
- 패키지·가상환경·실행 관리는 **`uv`**.
- 의존성은 `pyproject.toml`에 선언하고 잠금은 `uv.lock`으로 관리한다.
- 새 의존성은 `uv add 패키지명`, 개발 의존성은 `uv add --dev 패키지명`.
- 실행은 `uv run ...` 형태로 적는다.

### 2.2 금액 계산

- 모든 금액 계산은 `float`가 아니라 **`Decimal`** 을 사용한다.
- 금액, 가격, 수량, 수수료율은 내부에서 `Decimal`로 다루고, JSON 이벤트에는 문자열로 직렬화한다.
- 수수료는 체결금액 기준으로 계산하고, 매수는 취득 비용에 포함, 매도는 매도 수익에서 차감한다.
- 전략 성과·점수·학습은 **수수료 차감 후 순손익**으로 본다. 수수료 차감 전 수치도 함께 기록하되 판단 기준은 아니다.

### 2.3 재현성

- 모든 실험은 `코드 버전 + 데이터 기간 + 설정 + 점수식 + 탈락 이유`가 남아야 한다.
- 전략 버전이 바뀌면 이전 성과와 분리 집계한다.
- 백테스트는 해당 기간의 수수료율(`fee_rates`의 `effective_from`/`effective_to`)을 사용한다.

### 2.4 실패를 정상 경로로

다음은 예외가 아니라 기본 시나리오다. 코드에 반드시 경로가 있어야 한다.

- WebSocket 끊김·재연결
- REST API rate limit, 5xx 오류
- 주문 제출 후 응답 불명
- 부분 체결, 체결 지연
- 중복 이벤트, 시퀀스 누락
- 잔고 불일치 (reconciliation mismatch)
- DB 쓰기 실패
- 텔레그램 전송 실패

구체 절차는 [docs/operations.md](./docs/operations.md#장애-대응)를 본다.

---

## 3. 권한 경계

에이전트와 컴포넌트는 아래 표의 권한만 갖는다. 표에 없는 권한은 기본적으로 없다.

> **MVP 적용 범위**: `*(post-MVP)*` 태그가 붙은 행(Learning Loop 등)은 **지금 구현하지 않는다**. 태그가 없는 모든 행은 MVP에 포함된다. MVP의 실험 기능은 수동 backtest 실행과 결과 기록까지만 포함한다.

| 컴포넌트 | 만들 수 있는 것 | 만들 수 없는 것 |
|---|---|---|
| Momentum / Mean Reversion / Regime Filter Agent | `Signal` | `OrderRequest`, `RiskDecision` |
| Volatility Risk Agent *(post-MVP)* | `RiskSignal` | `OrderRequest` |
| Signal Aggregator | `PortfolioIntent` | `OrderRequest`, `RiskDecision` |
| Portfolio Manager | `ProposedTrade` | `OrderRequest`, `RiskDecision` |
| **Risk Gate** | `RiskDecision` (APPROVED/REJECTED/REDUCED/PAUSED) | `OrderRequest` |
| **Execution Agent** | **`OrderRequest`**, `OrderState`, 거래소 API 호출 | 리스크 룰 변경 |
| Backtest Runner / Paper Broker | 가상 체결, 실험 결과 | 실거래 주문 API 호출, 운영 API 키 접근 |
| Reconciliation Service | `PositionSnapshot`, `BalanceSnapshot` | `OrderRequest` |
| Fee Service | `FeeEvent`, 예상 수수료 | `OrderRequest`, `fee_rates` 자동 수정 |
| Agent Orchestrator | `WorkflowRun`, `WorkflowStep`, trace | `OrderRequest`, `RiskDecision`, 신호 임의 수정 |
| Strategy Lab / 실험 에이전트 *(post-MVP)* | `ExperimentProposal`, backtest 결과 | 실거래 주문 경로 접근, API 키 접근 |
| Learning Loop *(post-MVP)* | `LearningProposal` | live 코드/리스크 한도 직접 변경 |
| Approval Gate | 승인/거절 기록 | 제안 내용 수정 |
| Research / LLM 에이전트 *(post-MVP)* | `ContextSignal` (참고용) | `OrderRequest`, 포지션 크기 결정 |

Agent Orchestrator는 워크플로우 **흐름만** 제어한다. 투자 판단을 하지 않는다.

Learning Loop는 post-MVP에서 도입된다. 도입되면 학습 결과는 `LearningProposal`로 저장되고, Approval Gate를 통과해야만 반영된다. 자동 승격은 꺼둔다.

---

## 4. 언어와 문체 규칙

### 4.1 한국어가 기본이다

- 모든 문서, 주석, docstring, 운영 가이드, 에러 메시지는 **한국어**로 작성한다.
- 대상 독자는 "금융이나 컴퓨터를 전공하지 않았지만 이 시스템을 직접 운영해야 하는 사람"이다.
- 어려운 용어를 처음 쓸 때는 한 줄 설명을 붙인다.
  - 예: "슬리피지(주문을 넣은 가격과 실제 체결 가격의 차이)"
- 한 문장에 한 가지 뜻만 담는다. 짧고 구체적으로 쓴다.

### 4.2 코드 주석

- 주석은 **"무엇을 하는지"가 아니라 "왜 필요한지"** 를 설명한다.
- 단순한 코드에는 주석을 붙이지 않는다.
- 운영자가 알아야 하는 위험한 동작에는 조건과 결과를 분명히 적는다.

---

## 5. 코드 스타일

Python 코드는 [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)를 따른다. 설명은 한국어로, 형식은 Google 규칙으로.

### 5.1 기본

- 들여쓰기는 공백 4칸.
- 함수·메서드·변수·파일: `lower_with_under`
- 클래스: `CapWords`
- 상수: `CAPS_WITH_UNDER`
- 한 줄은 가능하면 80자 안.
- import 순서: 표준 라이브러리 → 서드파티 → 내부 모듈.
- 타입 힌트를 사용한다.
- 가변 기본값을 인자로 쓰지 않는다 (`items=[]` 금지).
- `except Exception` 남용 금지. 구체적인 예외만 잡는다.
- 로그는 lazy formatting: `logger.info("주문 ID: %s", order_id)`.

### 5.2 Docstring (Google 스타일, 한국어)

- `"""` 세 개 큰따옴표 사용.
- 요약 줄은 한 줄, 80자 이하, 마침표/물음표/느낌표 중 하나로 끝낸다.
- 설명이 더 필요하면 빈 줄 하나 두고 이어서 쓴다.
- 함수 인자는 `Args:`, 반환값은 `Returns:`, 예외는 `Raises:`, 클래스 공개 속성은 `Attributes:`.

좋은 예:

```python
def calculate_slippage_bps(expected_price: Decimal, filled_price: Decimal) -> Decimal:
    """주문 예상 가격과 실제 체결 가격의 차이를 bp 단위로 계산한다.

    bp는 0.01%를 뜻한다. 예를 들어 10bp는 0.10%다.
    이 값이 커지면 생각한 가격보다 불리하게 체결되었다는 뜻이다.

    Args:
        expected_price: 주문을 넣을 때 예상한 가격.
        filled_price: 거래소에서 실제로 체결된 가격.

    Returns:
        예상 가격과 실제 체결 가격의 차이. 단위는 bp다.

    Raises:
        ValueError: 예상 가격이 0보다 작거나 같을 때 발생한다.
    """
```

피해야 할 예:

```python
def calculate_slippage_bps(expected_price: Decimal, filled_price: Decimal) -> Decimal:
    """Calculate slippage in basis points."""  # 영어, 인자/반환 설명 없음
```

### 5.3 도구

- 포맷·린트: `uv run ruff check`, `uv run ruff format`
- 타입: `uv run pyright` 또는 `uv run mypy`
- 테스트: `uv run pytest`

---

## 6. 파일과 디렉토리 규약

- 저장소 루트 구조는 [docs/architecture.md](./docs/architecture.md#저장소-구조)를 따른다.
- 새 컴포넌트를 추가할 때는 architecture.md의 해당 섹션도 같이 갱신한다.
- 규칙·금지·권한 변경은 이 파일(AGENTS.md)에만 기록한다. 다른 문서에 복제하지 않는다.
- 운영 정책(리스크 한도, 장애 대응, 승인 채널)은 [docs/operations.md](./docs/operations.md)에만 기록한다.

---

## 7. 커밋·PR 전 체크리스트

코드를 올리기 전에 다음을 확인한다.

- [ ] `uv run ruff check`와 `uv run ruff format --check` 통과.
- [ ] `uv run pyright` 또는 `uv run mypy` 통과.
- [ ] `uv run pytest` 통과.
- [ ] 새 공개 함수·클래스에 Google 스타일 한국어 docstring이 있다.
- [ ] 금액 계산에 `float`를 쓰지 않았다.
- [ ] 새 의존성이 있다면 `pyproject.toml`과 `uv.lock`이 갱신됐다.
- [ ] API 키, secret, `.env`가 커밋에 섞여 있지 않다.
- [ ] 3절의 권한 경계를 어기지 않았다.
- [ ] 이 변경이 `OrderRequest`/`RiskDecision`/`fee_rates`를 수정하는 경로라면 Approval Gate 통과 기록이 있다.
