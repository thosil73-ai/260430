# Product Requirements Document (PRD)
## 멜론 스마트팜 웹서비스 프로젝트

---

## 1. 프로젝트 개요

### 1.1 프로젝트명
멜론 스마트팜 웹서비스 (Melon SmartFarm Web Service)

### 1.2 프로젝트 목적
아두이노 기반 하드웨어와 웹 대시보드를 연동하여 멜론 재배 환경을 실시간으로 모니터링하고, MOSFET 기반 관수 펌프를 안전하게 원격 제어하며, 수확 전 단수(물 끊기) 로직을 표준화하여 당도 및 품질 편차를 줄이는 것을 목표로 한다.

### 1.3 프로젝트 범위
- 하드웨어: Arduino Uno R4 WiFi 기반 센서/액츄에이터 제어
- 통신: MQTT 기반 실시간 양방향 통신(센서 Publish, 제어 Subscribe)
- 백엔드: Next.js API Routes + 규칙 엔진 + MQTT 브리지
- 프론트엔드: Next.js App Router 기반 대시보드/설정/이력 화면
- 데이터베이스: Supabase(PostgreSQL + Realtime + RLS)
- 운영기능: 경보 알림, 감사 로그, 단수 캘린더, 장치 헬스체크

### 1.4 배경 및 문제 정의
- 기존 농가 운영은 경험 기반 수동 제어 비중이 높아, 환경 급변 대응이 늦고 기록 일관성이 낮다.
- 관수 타이밍 관리가 사람 의존적이라 과다/과소 관수 리스크가 존재한다.
- 수확 전 단수 시점이 표준화되지 않아 품질 편차가 발생한다.
- 하드웨어 제어 실패(통신/전원/릴레이 불량) 시 즉각적인 원인 추적이 어렵다.

### 1.5 제품 목표 및 KPI

#### 제품 목표
1. 하우스 환경 상태(온도/습도/EC/pH)를 5초 단위로 실시간 가시화한다.
2. MOSFET 기반 펌프 제어를 수동/자동/예외 정책으로 일관 운영한다.
3. 수확 전 단수 상태머신을 적용하여 자동 차단과 예외 승인 프로세스를 제공한다.
4. 운영자가 현장에서 모바일로도 즉시 의사결정 가능한 UI를 제공한다.

#### KPI
- 센서 데이터 수집 성공률 99% 이상(일 단위)
- 펌프 제어 명령 성공률 99% 이상(ACK 수신 기준)
- 단수 스케줄 준수율 95% 이상
- 이상 상황 탐지 후 사용자 알림 전파 10초 이내
- 월간 서비스 가용성 99.5% 이상

---

## 2. 시스템 아키텍처

### 2.1 전체 아키텍처

```text
[Arduino Uno R4 + Sensor/MOSFET Driver]
        |  (MQTT Publish/Subscribe over WiFi)
        v
[MQTT Broker: Mosquitto]
        |  (Bridge)
        v
[Next.js Backend(API Routes + Rule Engine + MQTT Client)]
        |                          |
        |                          +--> [Alert Engine]
        v
[Supabase(PostgreSQL + Realtime + RLS)]
        ^
        |
[Next.js Frontend Dashboard/Settings/History]
```

### 2.2 레이어별 기술 스택

#### 하드웨어 레이어
- 보드: Arduino Uno R4 WiFi
- 언어: C++(Arduino)
- 라이브러리:
  - `PubSubClient` (MQTT)
  - `DHT` 라이브러리
  - `ArduinoJson`
- 센서:
  - 온도/습도(DHT22)
  - EC(아날로그)
  - pH(아날로그)
- 액츄에이터:
  - LED(PWM)
  - 양액 펌프(MOSFET 구동)
  - 팬1/팬2(릴레이 또는 MOSFET 확장 가능)

#### 백엔드 레이어
- 런타임: Node.js 18+ LTS
- 프레임워크: Next.js 14+ (App Router)
- 핵심 패키지:
  - `mqtt`
  - `@supabase/supabase-js`
  - `zod`(요청 검증)

#### 데이터 레이어
- Supabase PostgreSQL
- Realtime 구독
- RLS 정책 기반 접근 제어

#### 프론트엔드 레이어
- Next.js 14+ / React 18+ / TypeScript
- Tailwind CSS / shadcn/ui
- Recharts(시계열 시각화)

### 2.3 운영 환경
- 개발: 로컬 Mosquitto + Supabase Cloud
- 운영: 클라우드 Next.js(Vercel 또는 자체 서버) + 관리형 DB
- 모니터링: API 로그, MQTT 연결 상태, 장치 heartbeat

---

## 3. 기능 요구사항

### 3.1 센서 모니터링

#### 3.1.1 센서 사양
- 온도(DHT22): -40~80°C, 정확도 ±0.5°C
- 습도(DHT22): 0~100% RH, 정확도 ±2% RH
- EC: 0~20 mS/cm
- pH: 0~14 pH

#### 3.1.2 데이터 수집/전송
- 기본 수집 주기: 5초
- 전송 포맷: JSON
- 전송 방식: MQTT Publish
- 센서 유효성 검증:
  - 범위 초과값은 폐기 또는 `invalid` 플래그 저장
  - 연속 3회 실패 시 센서 장애 이벤트 발생

#### 3.1.3 화면 표시
- 실시간 카드(최근값/변화율/상태색)
- 차트 기간: 1시간, 24시간, 7일, 사용자 지정
- 임계값 라인 표시(상한/하한)
- 데이터 누락 구간 시각적 표시(점선 또는 음영)

### 3.2 액츄에이터 제어

#### 3.2.1 제어 대상
- LED: PWM 밝기 0~100%
- 양액 펌프: ON/OFF, 추후 PWM 유량 확장 가능
- 팬1/팬2: ON/OFF

#### 3.2.2 제어 모드
- 수동 제어: 운영자가 즉시 제어
- 자동 제어: 규칙 엔진 결과로 제어
- 안전 제어: 통신 이상 시 Fail-safe OFF

#### 3.2.3 이력 관리
- 누가, 언제, 어떤 장치를, 어떤 값으로 제어했는지 저장
- 명령 요청과 장치 ACK를 분리 저장
- 실패 사유 코드 저장(`TIMEOUT`, `DEVICE_OFFLINE`, `SAFETY_LOCK`)

### 3.3 수확 전 단수(물 끊기) 로직

#### 3.3.1 단수 정책 핵심
- 기준일: `예상 수확일 - 단수일수(N일)`
- 단수 시작 후 자동 관수는 전면 차단
- 예외 관수는 관리자 승인 + 사유 입력 필수

#### 3.3.2 상태머신 정의
- `NORMAL`: 일반 운영
- `CUTOFF_SCHEDULED`: 단수 예정
- `CUTOFF_ACTIVE`: 단수 진행(자동 관수 차단)
- `CUTOFF_OVERRIDE`: 예외 관수 일시 허용
- `CUTOFF_ENDED`: 수확 완료 또는 수동 종료

#### 3.3.3 상태 전이 규칙
1. `NORMAL -> CUTOFF_SCHEDULED`
   - 예상 수확일/단수일수 입력 시
2. `CUTOFF_SCHEDULED -> CUTOFF_ACTIVE`
   - 시작 조건 시간 도달 시 자동 전이
3. `CUTOFF_ACTIVE -> CUTOFF_OVERRIDE`
   - 관리자 승인 + 사유 입력 + 허용 시간 설정
4. `CUTOFF_OVERRIDE -> CUTOFF_ACTIVE`
   - 허용 시간 만료 또는 수동 종료
5. `CUTOFF_ACTIVE -> CUTOFF_ENDED`
   - 수확 완료 처리 또는 긴급 해제

#### 3.3.4 단수 중 제어 제한
- 자동 관수 명령: 차단
- 수동 관수 명령: `OWNER`, `ADMIN`만 허용
- 예외 관수 실행 시 필수 로그:
  - 사용자 ID
  - 사유
  - 관수 시간
  - 종료 시각

### 3.4 알림 및 이벤트
- 이벤트 레벨: `INFO`, `WARN`, `ALERT`, `CRITICAL`
- 기본 알림 채널: 웹 대시보드 인앱 알림
- 확장 채널: SMS/카카오/슬랙 Webhook
- 중복 억제: 동일 이벤트 10분 재발생 시 집계 알림
- 미확인 경고 누적 건수 배지 표시

### 3.5 대시보드 UX
- `/dashboard`:
  - 실시간 센서 카드
  - 제어 패널
  - 단수 상태 배너
  - 이벤트 타임라인
- `/settings`:
  - 임계치 설정
  - 자동 제어 규칙
  - 단수 캘린더/기본값
- `/history`:
  - 센서/제어/알림/감사로그 통합 검색

---

## 4. 하드웨어 요구사항 (MOSFET 중심)

### 4.1 펌프 구동 회로 요구사항
- N채널 로직레벨 MOSFET 사용(예: IRLZ44N 계열)
- 게이트 저항(100~220옴 권장)
- 게이트 풀다운 저항(10k옴 권장)
- 플라이백 다이오드(펌프 병렬 역방향 연결)
- MCU GND와 외부 전원 GND 공통 접지
- 펌프 전원은 MCU 전원과 분리 공급 권장

### 4.2 안전 요구사항
- 최대 연속 동작 제한(예: 600초)
- 제어 쿨다운(예: OFF 후 120초)
- 일일 누적 동작시간 상한(예: 90분)
- 통신 끊김/장치 재부팅 시 기본 OFF
- 과전류 감지(옵션) 시 즉시 차단 + 경고 발행

### 4.3 핀맵(초안)
- D2: DHT22
- A0: EC
- A1: pH
- D3(PWM): LED
- D4: Pump MOSFET Gate
- D5: Fan1
- D6: Fan2

---

## 5. MQTT 설계

### 5.1 토픽 규칙
- 접두사: `smartfarm/{farmId}/{deviceId}/...`
- 예시:
  - `smartfarm/farm-01/uno-r4-01/sensors/all`
  - `smartfarm/farm-01/uno-r4-01/actuators/pump/cmd`
  - `smartfarm/farm-01/uno-r4-01/actuators/pump/ack`
  - `smartfarm/farm-01/uno-r4-01/status`

### 5.2 QoS/Retain 정책
- 센서 데이터: QoS 0, Retain false
- 제어 명령: QoS 1, Retain false
- 장치 상태(online/offline): QoS 1, Retain true

### 5.3 메시지 페이로드 예시

```json
{
  "trace_id": "b95f9b5b-0df7-4e42-8c91-7f31a6c4c4a1",
  "type": "pump_command",
  "state": "ON",
  "duration_sec": 120,
  "requested_by": "user-uuid",
  "requested_at": "2026-04-30T06:10:20Z"
}
```

### 5.4 재전송 정책
- ACK 미수신 3초 경과 시 1회 재전송
- 최대 3회 재시도 후 실패 처리
- 실패 시 API 응답과 알림 이벤트 동시 생성

---

## 6. 백엔드/API 요구사항

### 6.1 API 목록
- `GET /api/sensors/latest`
- `GET /api/sensors?type=&from=&to=`
- `POST /api/actuators/pump`
- `POST /api/cutoff/start`
- `POST /api/cutoff/override`
- `POST /api/cutoff/stop`
- `GET /api/events`
- `GET /api/devices/health`

### 6.2 API 공통 규칙
- 요청/응답에 `trace_id` 포함
- 서버 시간 UTC 기준 저장
- 입력 검증 실패 시 400 + 상세 필드 오류 반환
- 제어 API는 멱등키 헤더(`Idempotency-Key`) 지원

### 6.3 제어 API 응답 예시

```json
{
  "trace_id": "b95f9b5b-0df7-4e42-8c91-7f31a6c4c4a1",
  "accepted": true,
  "command_id": 1021,
  "status": "PENDING_ACK",
  "message": "제어 명령이 전송되었습니다."
}
```

---

## 7. 데이터베이스 설계 (Supabase/PostgreSQL)

### 7.1 핵심 테이블
- `sensor_data`
  - `id`, `farm_id`, `zone_id`, `device_id`, `sensor_type`, `value`, `unit`, `quality`, `created_at`
- `actuator_commands`
  - `id`, `farm_id`, `device_id`, `actuator_type`, `action`, `value`, `source`, `trace_id`, `requested_by`, `requested_at`
- `actuator_acks`
  - `id`, `command_id`, `ack_state`, `ack_at`, `error_code`, `raw_payload`
- `cutoff_schedules`
  - `id`, `farm_id`, `zone_id`, `harvest_date`, `cutoff_days`, `state`, `started_at`, `ended_at`
- `cutoff_overrides`
  - `id`, `schedule_id`, `approved_by`, `reason`, `duration_sec`, `started_at`, `ended_at`
- `system_events`
  - `id`, `farm_id`, `level`, `event_type`, `message`, `ack_by`, `ack_at`, `created_at`
- `audit_logs`
  - `id`, `farm_id`, `user_id`, `action`, `target`, `trace_id`, `meta`, `created_at`

### 7.2 인덱스 전략
- `sensor_data(device_id, created_at DESC)`
- `sensor_data(sensor_type, created_at DESC)`
- `actuator_commands(trace_id)`
- `system_events(level, created_at DESC)`

### 7.3 데이터 보존 정책
- 원시 센서 데이터: 12개월
- 1분 집계 데이터: 36개월
- 제어/감사 로그: 36개월 이상

### 7.4 RLS 정책 개요
- 농장(`farm_id`) 단위 데이터 격리
- `viewer`: 조회만 가능
- `operator`: 센서 조회 + 일반 제어
- `admin`: 단수 오버라이드/설정 변경/권한 관리

---

## 8. 자동 제어 규칙 엔진

### 8.1 규칙 우선순위
1. 안전 규칙(과전류, 타임아웃, 통신오류)
2. 단수 규칙
3. 수동 강제 제어
4. 자동 관수 규칙

### 8.2 규칙 예시
- 규칙 A: 습도 하한 미만 + 단수 비활성 + 쿨다운 종료 -> 펌프 ON 60초
- 규칙 B: 온도 상한 초과 -> 팬1 ON
- 규칙 C: 단수 활성 -> 모든 자동 관수 차단

### 8.3 충돌 해결
- 동일 장치 충돌 시 높은 우선순위 규칙만 실행
- 차단된 규칙도 `skipped_reason`으로 이력 저장

---

## 9. 비기능 요구사항

### 9.1 성능
- 센서 반영 지연: 2초 이내(브로커 수신 후 UI 표시)
- 제어 왕복 지연: 평균 3초 이내(명령~ACK)
- 페이지 초기 로딩: 2초 이내(대시보드 기준)

### 9.2 안정성
- MQTT 자동 재연결
- WiFi 재연결 및 로컬 버퍼링 후 재전송
- 서버 장애 시 마지막 안전상태 유지(펌프 OFF)

### 9.3 보안
- MQTT 브로커 인증 계정 분리(장치/서버)
- TLS 적용 가능 구조(운영 필수 권장)
- 비밀키/토큰은 환경변수 관리
- 중요 제어(단수 해제, 오버라이드)는 재인증 또는 2차 확인

### 9.4 확장성
- 멀티 농장/멀티 디바이스 확장 고려
- 센서/액츄에이터 타입 추가 시 스키마 확장 최소화

---

## 10. 프로젝트 구조(권장)

```text
SmartFarmWeb/
├─ arduino/
│  └─ smartfarm_uno_r4/
│     ├─ smartfarm_uno_r4.ino
│     ├─ sensors.(h|cpp)
│     ├─ actuators.(h|cpp)
│     ├─ safety.(h|cpp)
│     └─ mqtt_client.(h|cpp)
├─ web/
│  ├─ src/app/
│  │  ├─ dashboard/
│  │  ├─ settings/
│  │  ├─ history/
│  │  └─ api/
│  ├─ src/components/
│  ├─ src/lib/
│  └─ src/types/
├─ mqtt/
│  ├─ mosquitto.conf
│  └─ topics.md
├─ supabase/
│  ├─ migrations/
│  └─ seed.sql
└─ docs/
   ├─ prd.md
   ├─ architecture.md
   ├─ api.md
   └─ hardware.md
```

---

## 11. 개발 단계

### Phase 1. 기반 구축 (2주)
- 프로젝트 초기 세팅(Arduino/Web/Supabase)
- MQTT 토픽/페이로드 표준 확정
- 기본 센서 수집 및 저장

### Phase 2. 제어 기능 (2~3주)
- MOSFET 펌프 제어 구현
- ACK/재시도/실패 처리
- 제어 이력 및 감사 로그 저장

### Phase 3. 단수/자동화 (2주)
- 단수 상태머신 구현
- 자동 관수 규칙 엔진
- 오버라이드 승인 플로우

### Phase 4. 운영 고도화 (2주)
- 알림/이벤트 대시보드
- 장치 헬스체크/장애 대응
- 성능 튜닝 및 통합 테스트

---

## 12. 테스트 전략

### 12.1 단위 테스트
- 규칙 엔진 우선순위
- 단수 상태 전이
- API 검증 스키마

### 12.2 통합 테스트
- 센서 Publish -> DB 저장 -> UI 반영
- 제어 API -> MQTT 명령 -> 장치 ACK -> 이력 저장
- 단수 활성 시 자동 관수 차단 확인

### 12.3 하드웨어 테스트
- MOSFET 발열 및 구동 안정성
- 플라이백 다이오드 유무 비교 테스트
- 전원 강하/노이즈 상황 재현 테스트

### 12.4 장애 테스트
- WiFi 단절
- MQTT 브로커 중단
- DB 지연/장애
- 장치 재부팅/오프라인

---

## 13. 운영 시나리오(핵심 플로우)

### 13.1 일상 운영
1. 센서 데이터 5초 주기 수집
2. 임계치 초과 시 대시보드 경고
3. 필요 시 운영자가 수동 제어

### 13.2 단수 시작 시점
1. 수확 예정일 등록
2. 단수 예정일 도달 시 자동 `CUTOFF_ACTIVE`
3. 자동 관수 차단 + 단수 시작 알림 발송

### 13.3 단수 중 예외 관수
1. 관리자만 오버라이드 요청 가능
2. 사유 입력 및 허용 시간 설정
3. 시간 만료 시 자동 복귀 + 로그 저장

---

## 14. 제약사항 및 가정

### 14.1 제약사항
- Arduino Uno R4 WiFi 네트워크 품질에 의존
- 센서(EC/pH) 보정 주기 준수 필요
- 펌프 전원 설계 미흡 시 제어 오동작 가능

### 14.2 가정
- 운영자는 기본적인 장치 설치/교체 가능
- 하우스별 수확 일정 입력 프로세스가 존재
- 관리자 계정은 현장별 최소 1인 이상 보유

---

## 15. 리스크 및 대응

- 센서 드리프트: 정기 캘리브레이션 알림 제공
- 오제어 위험: 안전규칙 우선 + 수동 긴급정지 버튼
- 단수 과도 적용: 품종별 기본 단수일수 템플릿 제공
- 네트워크 불안정: 로컬 큐/재전송 + 오프라인 표시
- 운영 실수: 권한 기반 UI 가드 + 확인 모달 + 감사로그

---

## 16. 승인 기준 (Definition of Done)

### 16.1 기능 완료 기준
- 실시간 센서 모니터링/차트 정상 동작
- MOSFET 펌프 수동/자동 제어 동작
- 단수 상태머신 및 예외 관수 정책 동작
- 이력/감사로그/알림 기능 동작

### 16.2 품질 완료 기준
- 핵심 E2E 시나리오 10건 이상 통과
- 제어 실패 재시도/에러표시 검증 통과
- 보안 점검(RLS/비밀키 노출 방지) 완료

### 16.3 문서 완료 기준
- 설치 가이드, 핀맵, 운영 매뉴얼 제공
- MQTT 토픽 명세와 API 명세 최신화 완료

---

## 17. 참고 문서
- [Arduino Uno R4 공식 문서](https://docs.arduino.cc/)
- [Next.js 공식 문서](https://nextjs.org/docs)
- [Supabase 공식 문서](https://supabase.com/docs)
- [MQTT 공식 사이트](https://mqtt.org/)
- [Mosquitto 공식 문서](https://mosquitto.org/documentation/)

