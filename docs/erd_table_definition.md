# CoLiving IoT 플랫폼 — ERD & 테이블 정의서 (초안)

> [!NOTE]
> 기획안 섹션 7(도메인 모델)을 기반으로 작성한 **초안**입니다. 멘토 검토 후 확정하세요.

---

## 1. ERD (Entity Relationship Diagram)

```mermaid
erDiagram
    TENANT ||--o{ SITE : "보유"
    TENANT ||--o{ OPERATOR : "소속"
    SITE ||--o{ SPACE : "포함"
    SPACE ||--o{ DEVICE : "설치됨"
    SPACE ||--o{ CONTRACT : "임대 대상"
    SPACE ||--o{ RESERVATION : "예약 대상"
    RESIDENT ||--o{ CONTRACT : "계약 체결"
    RESIDENT ||--o{ RESERVATION : "예약 신청"
    DEVICE ||--o{ CONTROL_LOG : "제어 이력"
    OPERATOR ||--o{ CONTROL_LOG : "기록 주체 (관리자)"
    RESIDENT ||--o{ CONTROL_LOG : "기록 주체 (입주자)"

    TENANT {
        bigint tenant_id PK
        varchar name
        varchar business_number
        varchar contact_email
        varchar contact_phone
        timestamp created_at
        timestamp updated_at
    }

    SITE {
        bigint site_id PK
        bigint tenant_id FK
        varchar name
        varchar address
        varchar region
        timestamp created_at
    }

    SPACE {
        bigint space_id PK
        bigint site_id FK
        varchar name
        varchar space_type "PRIVATE | COMMON"
        varchar status "AVAILABLE | OCCUPIED | MAINTENANCE"
        int floor
        numeric area_sqm
        int max_capacity "공용 공간 전용 (nullable)"
        varchar operating_hours "공용 공간 전용 (nullable)"
        boolean is_reservable "공용 공간 전용"
        timestamp created_at
    }

    OPERATOR {
        bigint operator_id PK
        bigint tenant_id FK
        varchar login_id
        varchar password_hash
        varchar name
        varchar role "ADMIN | MANAGER"
        timestamp created_at
    }

    RESIDENT {
        bigint resident_id PK
        varchar login_id
        varchar password_hash
        varchar name
        varchar phone
        varchar email
        timestamp created_at
    }

    CONTRACT {
        bigint contract_id PK
        bigint resident_id FK
        bigint space_id FK
        date start_date
        date end_date
        varchar status "ACTIVE | EXPIRED | TERMINATED"
        numeric monthly_rent
        numeric deposit
        timestamp created_at
    }

    DEVICE {
        bigint device_id PK
        bigint space_id FK
        varchar device_name
        varchar device_type "LIGHT | HVAC | LOCK | WASHER"
        varchar model
        varchar status "ONLINE | OFFLINE | ERROR"
        varchar mock_endpoint "목업 IoT 서버 URL"
        boolean is_active
        timestamp installed_at
    }

    RESERVATION {
        bigint reservation_id PK
        bigint resident_id FK
        bigint space_id FK
        date reservation_date
        time start_time
        time end_time
        varchar status "PENDING | APPROVED | CANCELLED"
        timestamp created_at
    }

    CONTROL_LOG {
        bigint log_id PK
        bigint device_id FK
        bigint actor_id "조작한 사용자 ID"
        varchar actor_type "OPERATOR | RESIDENT"
        varchar command "TURN_ON | TURN_OFF | SET_TEMP 등"
        varchar request_payload
        varchar response_payload
        varchar result "SUCCESS | FAIL"
        timestamp executed_at
    }
```

---

## 2. 테이블 정의서

### 2.1 TENANT (운영사)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `tenant_id` | BIGINT | PK | NO | 운영사 고유 ID |
| `name` | VARCHAR(100) | | NO | 운영사명 |
| `business_number` | VARCHAR(20) | | NO | 사업자등록번호 |
| `contact_email` | VARCHAR(255) | | YES | 대표 이메일 |
| `contact_phone` | VARCHAR(20) | | YES | 대표 전화번호 |
| `created_at` | TIMESTAMP | | NO | 등록일시 |
| `updated_at` | TIMESTAMP | | YES | 수정일시 |

### 2.2 SITE (사이트 / 건물)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `site_id` | BIGINT | PK | NO | 사이트 고유 ID |
| `tenant_id` | BIGINT | FK → TENANT | NO | 소속 운영사 |
| `name` | VARCHAR(100) | | NO | 사이트명 (예: "강남점") |
| `address` | VARCHAR(255) | | NO | 주소 |
| `region` | VARCHAR(50) | | YES | 지역 분류 |
| `created_at` | TIMESTAMP | | NO | 등록일시 |

### 2.3 SPACE (공간 — 호실 + 공용)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `space_id` | BIGINT | PK | NO | 공간 고유 ID |
| `site_id` | BIGINT | FK → SITE | NO | 소속 사이트 |
| `name` | VARCHAR(100) | | NO | 공간명 (예: "301호", "1층 세탁실") |
| `space_type` | VARCHAR(20) | | NO | **PRIVATE** (호실) / **COMMON** (공용) |
| `status` | VARCHAR(20) | | NO | AVAILABLE / OCCUPIED / MAINTENANCE |
| `floor` | INT | | YES | 층수 |
| `area_sqm` | NUMERIC(6,2) | | YES | 면적 (㎡) |
| `max_capacity` | INT | | YES | 최대 수용 인원 (**공용만**) |
| `operating_hours` | VARCHAR(50) | | YES | 운영 시간 (**공용만**, 예: "06:00-23:00") |
| `is_reservable` | BOOLEAN | | NO | 예약 가능 여부 (공용: true, 호실: false) |
| `created_at` | TIMESTAMP | | NO | 등록일시 |

> [!IMPORTANT]
> `space_type`으로 PRIVATE/COMMON을 구분합니다. 공용 전용 컬럼(`max_capacity`, `operating_hours`)은 PRIVATE일 때 NULL이 됩니다. MVP 이후 속성이 많이 분화되면 **서브타입 테이블 분리**를 검토하세요.

### 2.4 OPERATOR (운영 관리자)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `operator_id` | BIGINT | PK | NO | 관리자 고유 ID |
| `tenant_id` | BIGINT | FK → TENANT | NO | 소속 운영사 |
| `login_id` | VARCHAR(50) | UNIQUE | NO | 로그인 ID |
| `password_hash` | VARCHAR(255) | | NO | 암호화된 비밀번호 |
| `name` | VARCHAR(50) | | NO | 이름 |
| `role` | VARCHAR(20) | | NO | ADMIN / MANAGER |
| `created_at` | TIMESTAMP | | NO | 등록일시 |

### 2.5 RESIDENT (입주자)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `resident_id` | BIGINT | PK | NO | 입주자 고유 ID |
| `login_id` | VARCHAR(50) | UNIQUE | NO | 로그인 ID |
| `password_hash` | VARCHAR(255) | | NO | 암호화된 비밀번호 |
| `name` | VARCHAR(50) | | NO | 이름 |
| `phone` | VARCHAR(20) | | YES | 연락처 |
| `email` | VARCHAR(255) | | YES | 이메일 |
| `created_at` | TIMESTAMP | | NO | 등록일시 |

### 2.6 CONTRACT (계약)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `contract_id` | BIGINT | PK | NO | 계약 고유 ID |
| `resident_id` | BIGINT | FK → RESIDENT | NO | 입주자 |
| `space_id` | BIGINT | FK → SPACE | NO | 계약 대상 호실 |
| `start_date` | DATE | | NO | 계약 시작일 |
| `end_date` | DATE | | NO | 계약 종료일 |
| `status` | VARCHAR(20) | | NO | ACTIVE / EXPIRED / TERMINATED |
| `monthly_rent` | NUMERIC(10,0) | | NO | 월 임대료 (원) |
| `deposit` | NUMERIC(12,0) | | YES | 보증금 (원) |
| `created_at` | TIMESTAMP | | NO | 등록일시 |

### 2.7 DEVICE (기기)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `device_id` | BIGINT | PK | NO | 기기 고유 ID |
| `space_id` | BIGINT | FK → SPACE | NO | 설치된 공간 |
| `device_name` | VARCHAR(100) | | NO | 기기명 (예: "301호 천장 조명") |
| `device_type` | VARCHAR(20) | | NO | LIGHT / HVAC / LOCK / WASHER 등 |
| `model` | VARCHAR(100) | | YES | 모델명 |
| `status` | VARCHAR(20) | | NO | ONLINE / OFFLINE / ERROR |
| `mock_endpoint` | VARCHAR(255) | | YES | 목업 IoT 서버 호출 URL |
| `is_active` | BOOLEAN | | NO | 활성화 여부 |
| `installed_at` | TIMESTAMP | | NO | 설치일시 |

### 2.8 RESERVATION (예약)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `reservation_id` | BIGINT | PK | NO | 예약 고유 ID |
| `resident_id` | BIGINT | FK → RESIDENT | NO | 예약 신청 입주자 |
| `space_id` | BIGINT | FK → SPACE | NO | 예약 대상 공용 공간 |
| `reservation_date` | DATE | | NO | 예약 날짜 |
| `start_time` | TIME | | NO | 시작 시각 |
| `end_time` | TIME | | NO | 종료 시각 |
| `status` | VARCHAR(20) | | NO | PENDING / APPROVED / CANCELLED |
| `created_at` | TIMESTAMP | | NO | 등록일시 |

### 2.9 CONTROL_LOG (제어 이력)

| 컬럼 | 타입 | PK/FK | NULL | 설명 |
|---|---|---|---|---|
| `log_id` | BIGINT | PK | NO | 로그 고유 ID |
| `device_id` | BIGINT | FK → DEVICE | NO | 대상 기기 |
| `actor_id` | BIGINT | | NO | 제어한 사용자 ID |
| `actor_type` | VARCHAR(20) | | NO | OPERATOR / RESIDENT |
| `command` | VARCHAR(50) | | NO | 제어 명령 (TURN_ON, SET_TEMP 등) |
| `request_payload` | TEXT | | YES | 요청 본문 (JSON) |
| `response_payload` | TEXT | | YES | 응답 본문 (JSON) |
| `result` | VARCHAR(10) | | NO | SUCCESS / FAIL |
| `executed_at` | TIMESTAMP | | NO | 실행 시각 |

---

## 3. 주요 관계 요약

```mermaid
graph LR
    A["TENANT<br/>(운영사)"] -->|1 : N| B["SITE<br/>(건물)"]
    A -->|1 : N| C["OPERATOR<br/>(관리자)"]
    B -->|1 : N| D["SPACE<br/>(공간)"]
    D -->|1 : N| E["DEVICE<br/>(기기)"]
    D -->|1 : N| F["CONTRACT<br/>(계약)"]
    D -->|1 : N| G["RESERVATION<br/>(예약)"]
    H["RESIDENT<br/>(입주자)"] -->|1 : N| F
    H -->|1 : N| G
    E -->|1 : N| I["CONTROL_LOG<br/>(제어 이력)"]

    style A fill:#6366f1,color:#fff
    style B fill:#8b5cf6,color:#fff
    style D fill:#a78bfa,color:#fff
    style E fill:#f59e0b,color:#fff
    style H fill:#10b981,color:#fff
    style I fill:#ef4444,color:#fff
```

> [!TIP]
> **핵심 흐름:** Tenant → Site → Space → Device → ControlLog 가 IoT 제어의 메인 경로이고, Resident → Contract → Space 가 입주 관리의 메인 경로입니다.
