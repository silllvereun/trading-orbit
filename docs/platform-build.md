# ORBIT 플랫폼 구축 문서

## 1. 제품 방향

ORBIT는 Gate.io, Websea, Stockcoin을 시작으로 여러 거래소의 선물 API를 연결하고, 사용자별 자동매매 Worker를 운영하는 멀티 거래소 플랫폼이다.

- 브랜드 콘셉트: 우주 관제센터
- UX 원칙: **디자인만 ORBIT 콘셉트를 적용**하고, 화면 기능은 표준 용어를 사용한다.
- 화면 기능명: 대시보드 / 거래소 연결 / 전략 / 주문 / 포지션 / 로그 / 리스크 관리 / 설정
- 실거래 전 단계: 백테스트 → Paper Trading → 제한 운영 → 실거래

## 2. 핵심 모델

```
KYC 사용자
  └─ 거래소 계정 연결 (1:N)
       └─ API Key (1:N)
            └─ Worker 컨테이너 (API Key 1개당 1개)
                 ├─ 신호 판단 Job
                 ├─ 주문 실행 Job
                 └─ 포지션 모니터링 Job
```

### 설계 원칙

- API Key별 Worker, 포지션, 리스크 한도를 분리한다.
- Worker는 상태를 최소화하고, 주문·체결·포지션의 기준 데이터는 PostgreSQL에 저장한다.
- 모든 주문은 멱등 키(`clientOrderId`)와 DB 제약조건으로 중복 실행을 막는다.
- Worker 재시작 전 거래소의 포지션 및 미체결 주문을 동기화한 후 작업을 재개한다.

## 3. 거래소 연동

`ExchangeAdapter` 공통 인터페이스를 두고 거래소별 구현을 분리한다.

### 대상 거래소

- Gate.io
- Websea
- Stockcoin

### 필수 기능

- API Key 유효성 검증 및 잔고 조회
- 선물 심볼·최소 주문 단위·가격/수량 정밀도 동기화
- 시세·호가·캔들 데이터 수집 (WebSocket 우선, REST 폴백)
- 레버리지·마진모드 설정
- 시장가/지정가 주문, TP/SL 주문, 주문 취소·조회
- 포지션·미체결 주문·체결 내역 동기화
- Rate limit, 네트워크 오류, 거래소별 오류 코드 표준화
- 거래소 기능 차이(Native TP, WebSocket 등)는 capability로 관리

## 4. 자동매매 로직

### 4.1 신호 판단 Job

1. 거래량 상위 50개 심볼 후보를 갱신한다.
2. 최우선 호가의 명목가가 최소 100,000 USDT 이상인지 확인한다.
3. 통과 심볼의 OHLCV를 수집한다.
4. RSI, MACD, CCI, 거래량, 최근 고점(저항), 최근 저점(지지)을 계산한다.
5. 지표값, 캔들 시각, 전략 버전, 진입/제외 근거를 불변 로그로 저장한다.
6. 모든 조건이 충족될 때만 `ENTRY_APPROVED` 이벤트를 발행한다.

제외 사유도 반드시 저장한다.

- 호가 유동성 부족
- 이미 활성 포지션 존재
- 쿨다운 상태
- 리스크 한도 초과
- 거래소 데이터 지연 또는 장애
- 심볼 거래 불가

### 4.2 주문 실행 Job

`ENTRY_APPROVED` 이벤트 수신 후 다음을 재검증한다.

1. Worker/API Key 활성 상태
2. 전략 버전과 실행 시간
3. 쿨다운·일일 손실·최대 노출 한도
4. 심볼당 활성 포지션 존재 여부
5. 가용 증거금, 최소 주문금액, 수량·가격 정밀도

검증 통과 시 주문을 실행한다.

- 기본 레버리지: 20x
- 기본 목표: ROE +5% TP
- 실제 체결가와 체결 수량을 기준으로 포지션을 생성한다.
- 거래소 Native TP를 우선 사용하고, 불가 시 포지션 모니터가 동일하게 관리한다.
- 주문 요청/응답/거절 사유/거래소 주문 ID를 모두 저장한다.

### 4.3 포지션 모니터링 Job

활성 포지션에 대해 다음을 반복한다.

- 현재가, ROE, 미실현 PnL, 청산가, TP/SL 상태 갱신
- 진입 시점 지표와 최신 지표 비교
- 반대 방향의 종료 조건이 확정되면 시장가 청산
- TP 체결/수동 종료/강제청산/API 복구 종료 사유를 구분해 저장
- 거래소 데이터와 주기적으로 reconciliation 실행

## 5. 리스크 관리

### 요청 규칙

- 최근 손실 중 가장 큰 PnL 손실이 발생한 뒤, 해당 API Key의 신규 진입을 중지한다.
- 다음 4시간봉이 도래할 때까지 쿨다운한다.
- 쿨다운 시작·종료 시각, 대상 심볼, 트리거 거래를 기록하고 화면에 표시한다.

### MVP 필수 안전장치

- API Key별 최대 동시 포지션 수
- 심볼당 1개 활성 포지션
- API Key별 최대 증거금 및 총 노출 한도
- 일일 최대 손실, 연속 손실, 최대 낙폭 한도
- Hard Stop Loss / 최대 허용 ROE 손실
- 전체·거래소·사용자·API Key·Worker별 긴급 중지
- 데이터 지연, API 오류, 잔고 불일치, TP 등록 실패 시 신규 진입 중지
- 수동 청산 우선 및 관련 미체결 주문 취소
- 주문 전후 거래소 상태 재조회 및 분산 락

> 20배 레버리지에서 반대 신호만을 종료 기준으로 쓰면 급격한 변동 때 강제청산 위험이 있다. Hard Stop Loss는 선택이 아니라 필수다.

## 6. 데이터 모델

- `users`: KYC 사용자, 권한, 상태
- `exchange_accounts`: 사용자와 거래소 연결
- `api_credentials`: 마스킹된 Key 식별자, 암호화된 Secret, 권한, 활성 상태
- `trading_workers`: API Key별 Worker 상태, 헬스체크, 실행 전략
- `strategies`, `strategy_versions`: 전략과 버전별 파라미터
- `signals`: 심볼·시간봉·지표 스냅샷·판정 결과
- `orders`, `fills`, `positions`, `trade_results`: 주문·체결·포지션 생명주기
- `risk_events`, `cooldowns`, `audit_logs`: 리스크 및 감사 기록

권장 인덱스:

- `(worker_id, status)`
- `(api_credential_id, symbol, status)`
- `(event_type, created_at)`
- `(created_at)`

## 7. 대시보드 요구사항

### 대시보드

- Worker 상태: RUNNING / PAUSED / COOLDOWN / ERROR / STOPPED
- 거래소·사용자·API Key별 연결 수, 잔고, PnL
- 활성 포지션, 오늘의 실현/미실현 손익
- 최근 신호·주문·오류·리스크 이벤트
- 전체 및 단위별 긴급 중지

### 전략 및 신호

- 거래량 상위 50개 후보와 호가 조건 통과 여부
- 심볼·시간봉별 RSI/MACD/CCI/거래량/지지·저항 값
- LONG / SHORT / 진입 제외 결과와 판단 근거
- 진입 시점 스냅샷과 현재 데이터 비교

### 로그

모바일에서도 읽기 쉬운 콘솔형 단일 열 로그 UI를 만든다.

- 레벨: INFO / SIGNAL / ORDER / POSITION / RISK / ERROR
- 필터: 시간, 거래소, Worker, 심볼, 포지션 ID, 이벤트 타입, 레벨
- 기본 행: 사람이 읽는 요약
- 상세 보기: request ID, order ID, 지표값, 오류 코드 등 구조화 정보
- API Key, Secret, 개인식별정보는 항상 마스킹

## 8. 성능·신뢰성

- WebSocket을 가격·호가·체결 데이터의 기본 경로로 사용한다.
- REST는 초기 동기화, 주문, 폴백 용도로 사용한다.
- 심볼 메타데이터와 거래량 상위 후보는 Redis TTL 캐시로 운영한다.
- Signal과 주문 실행은 메시지 큐로 분리한다. 초기 구현은 Redis Streams를 우선 검토한다.
- Outbox 패턴과 이벤트 ID로 재시도 시에도 1회 실행을 보장한다.
- 구조화 JSON 로그, Trace ID, Health Check, 주문 지연/성공률/API 오류율 메트릭을 제공한다.
- Worker에는 liveness/readiness probe를 두고 장애 시 안전하게 재기동한다.

## 9. 보안

- 거래소 API Key는 거래 전용 권한만 허용하고 출금 권한을 금지한다.
- Secret은 KMS/Vault 기반 envelope encryption으로 암호화한다.
- Secret은 DB, 로그, 브라우저, 오류 메시지에 평문으로 남기지 않는다.
- 실제 환경값은 저장소가 아닌 Secret Manager/Vault에서 런타임 주입한다.
- 공개 저장소에는 `.env.example`만 둘 수 있으며, 실제 `.env`는 커밋 금지다.
- GitHub Secret Scanning과 Push Protection을 활성화한다.


### Row Level Security (RLS) 정책

PostgreSQL RLS를 적용하여 애플리케이션 실수나 API 취약점이 발생해도 다른 사용자의 API Key·포지션·거래 내역을 조회하거나 수정할 수 없게 한다.

- 테넌트 데이터 테이블에는 `user_id`를 필수로 두고, API Key 하위 데이터에는 `api_credential_id`와 `worker_id`를 함께 기록한다.
- RLS 적용 대상: `exchange_accounts`, `api_credentials`, `trading_workers`, `strategies`, `signals`, `orders`, `fills`, `positions`, `trade_results`, `risk_events`, `cooldowns`, `audit_logs`.
- 사용자 요청은 트랜잭션 시작 시 인증된 사용자 ID와 역할을 DB 세션 변수에 설정하고, `user_id = current_setting('app.user_id')` 조건의 정책으로 조회·수정 범위를 제한한다.
- 일반 애플리케이션 DB 역할에는 `BYPASSRLS` 권한을 부여하지 않는다.
- 관리자는 별도 관리자 역할과 감사 로그를 통해서만 범위가 확장된다.
- Worker는 자신에게 할당된 `worker_id`와 연결된 API Key 데이터만 접근하도록 별도 정책을 둔다.
- 관리자·Worker의 모든 범위 확장 조회/수정은 `audit_logs`에 남긴다.
- `SECURITY DEFINER` 함수는 최소화하고, 사용 시 검색 경로 고정·권한·입력값 검증을 명시한다.



## 10. Docker Image 기반 SSH 자동 배포

초기 운영 배포는 GitHub Actions가 Docker Image를 빌드하고, 검증 후 SSH로 운영 서버에 접속해 버전을 교체하는 방식으로 구성한다.

```
main 브랜치 Merge
  → CI 테스트·보안 검사
  → Docker Image Build
  → GitHub Container Registry(GHCR) Push
  → SSH 배포 서버 접속
  → Image SHA Pull
  → DB Migration
  → docker compose up -d
  → Health Check
  → 성공/실패 알림 및 필요 시 Rollback
```

### 배포 원칙

- `latest` 태그 대신 commit SHA 또는 release version의 **불변 Image 태그**로 배포한다.
- GitHub Actions는 테스트·빌드·이미지 푸시까지만 담당하고, 실제 Secret은 서버의 Secret Manager/Vault 또는 보호된 환경 파일에서 주입한다.
- SSH private key, 서버 주소, GHCR 인증 정보는 GitHub Actions Secrets에만 저장하며 로그에 출력하지 않는다.
- SSH는 host key 검증을 적용하고, 배포 전용 사용자에게 필요한 Docker 권한만 부여한다.
- `docker compose` 서비스에는 healthcheck를 정의하고, API/Worker가 정상 상태일 때만 배포 성공으로 처리한다.
- DB migration은 애플리케이션 시작 전 또는 별도 migration job으로 단일 실행한다. 여러 Worker가 동시에 migration을 실행하면 안 된다.
- 실패 시 직전 정상 Image SHA로 즉시 rollback할 수 있도록 배포 이력을 저장한다.
- Worker 재배포 전에는 신규 진입을 잠시 중지하고, 미체결 주문·활성 포지션을 거래소와 동기화한 뒤 재개한다.
- 운영 환경은 Nginx 또는 Load Balancer 뒤에 두고 HTTPS만 허용한다.

## 11. 권장 구현 순서

1. 공통 DB 스키마, 인증/권한, API Key 암호화
2. Gate.io 단일 거래소 어댑터 및 Paper Trading
3. 주문·체결·포지션 동기화와 멱등성 처리
4. 신호 판단 Job 및 지표·로그 저장
5. 리스크 엔진, 쿨다운, Hard SL, 긴급 중지
6. 반응형 대시보드와 로그 콘솔
7. Websea·Stockcoin 어댑터 및 capability 테스트
8. API Key 1:1 Worker 오케스트레이션과 모니터링
9. 백테스트·부하·장애복구·중복이벤트 테스트
10. 제한 운영 후 실거래 전환

## 12. 완료 기준

- [ ] 3개 거래소 API Key를 사용자별로 안전하게 여러 개 연결할 수 있다.
- [ ] API Key 1개당 독립 Worker가 기동·중지·복구된다.
- [ ] 상위 50개, 1호가 100k, 5개 지표 판단 결과를 저장·조회할 수 있다.
- [ ] 신호/주문/포지션 종료가 분리되고 중복 주문이 발생하지 않는다.
- [ ] 20x, TP +5% ROE, 반대 신호 종료, 4시간봉 쿨다운이 설정·로그에 반영된다.
- [ ] Hard SL, 노출/손실 한도, 긴급 중지 기능이 동작한다.
- [ ] PC와 모바일에서 대시보드와 로그를 읽고 안전하게 제어할 수 있다.
- [ ] Paper Trading, 재기동 복구, API 오류, 중복 이벤트, 부하 테스트를 통과한다.
- [ ] Docker Image SHA 기반 SSH 자동 배포·Health Check·Rollback이 동작한다.
- [ ] RLS 정책으로 사용자·API Key·Worker 데이터 경계가 DB 레벨에서 강제된다.
