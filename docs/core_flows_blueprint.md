# Core Flow Blueprints (핵심 비즈니스 로직 설계)

이 문서는 **Must 기능**을 중심으로 한 시스템의 핵심 동작 흐름을 시각화한 블루프린트입니다. 각 시퀀스 다이어그램은 프론트엔드(BFF), 백엔드(Spring), 외부 시스템(Mock IoT), 그리고 데이터베이스 간의 상호작용을 보여줍니다.

---

## 1. 인증 및 인가 플로우 (Login & Auth)
사용자가 로그인하여 권한에 맞는 대시보드로 진입하는 과정입니다.

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자 (Admin/Resident)
    participant FE as 프론트엔드 (Next.js/BFF)
    participant BE as 백엔드 (Spring Boot)
    participant DB as 데이터베이스 (PostgreSQL)

    User->>FE: ID/비밀번호 입력
    FE->>BE: POST /api/auth/login (LoginRequest)
    BE->>DB: 사용자 정보 및 역할(Role) 조회
    DB-->>BE: 사용자 정보 반환
    BE->>BE: 비밀번호 해시 검증
    BE->>BE: JWT (Access/Refresh Token) 생성
    BE-->>FE: JWT + User Info 응답
    FE->>FE: JWT를 Secure/HttpOnly 쿠키에 저장
    FE->>User: 역할(Role)에 따른 대시보드 리다이렉트
    Note over User, FE: ADMIN -> /admin/dashboard<br/>RESIDENT -> /my-room
```

---

## 2. 입주자 기기 제어 플로우 (Device Control)
입주자가 본인의 방에 있는 IoT 기기를 제어하고, 그 이력이 기록되는 과정입니다.

```mermaid
sequenceDiagram
    autonumber
    actor Resident as 입주자
    participant FE as 프론트엔드 (Next.js/BFF)
    participant BE as 백엔드 (Spring Boot)
    participant IoT as 목업 IoT 서버
    participant DB as 데이터베이스 (PostgreSQL)

    Resident->>FE: 기기 제어 버튼 클릭 (예: 조명 ON)
    FE->>BE: POST /api/resident/devices/{id}/control (Command)
    
    rect rgb(240, 240, 240)
        Note right of BE: 권한 검증 및 상태 확인
        BE->>DB: 계약(Contract) 및 공간(Space) 소유권 확인
        DB-->>BE: 권한 정보 반환
        BE->>DB: 기기 상태(Online/Offline) 확인
        DB-->>BE: 기기 정보 반환
    end

    alt 검증 통과
        BE->>IoT: HTTP POST (제어 명령 전송)
        IoT-->>BE: 200 OK (상태 변경 완료)
        BE->>DB: CONTROL_LOG 테이블에 이력 기록
        BE-->>FE: 성공 응답 및 변경된 상태 반환
        FE-->>Resident: UI 업데이트 (조명 아이콘 켜짐)
    else 권한 없음 또는 기기 오프라인
        BE-->>FE: 에러 응답 (403 or 409)
        FE-->>Resident: 알림 메시지 표시
    end
```

---

## 3. 관리자 운영 설정 플로우 (Admin Setup Flow)
관리자가 새로운 공간을 만들고, 기기를 배치하고, 입주자와 계약을 연결하는 초기 설정 과정입니다.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 관리자
    participant FE as 프론트엔드 (Next.js/BFF)
    participant BE as 백엔드 (Spring Boot)
    participant DB as 데이터베이스 (PostgreSQL)

    Note over Admin, DB: [Step 1] 공간(Space) 생성
    Admin->>FE: 공간 정보 입력 (301호, Private)
    FE->>BE: POST /api/admin/spaces
    BE->>DB: SPACE 테이블에 저장
    DB-->>BE: 저장 완료

    Note over Admin, DB: [Step 2] 기기(Device) 배치
    Admin->>FE: 기기 정보 입력 (조명, Mock Endpoint)
    FE->>BE: POST /api/admin/devices (space_id 포함)
    BE->>DB: DEVICE 테이블에 저장
    DB-->>BE: 저장 완료

    Note over Admin, DB: [Step 3] 계약(Contract) 체결
    Admin->>FE: 계약 정보 입력 (입주자A, 301호, 기간)
    FE->>BE: POST /api/admin/contracts
    BE->>DB: CONTRACT 생성 및 SPACE 상태 변경(AVAILABLE -> OCCUPIED)
    DB-->>BE: 완료
    BE-->>FE: 설정 완료 응답
    FE-->>Admin: "관리 준비 완료" 메시지 표시
```

---

## 4. 블루프린트 활용 가이드
- **개발 우선순위**: 위 흐름도(1->2->3) 순서대로 API 인터페이스를 확정(API Freeze)하고 개발을 진행하면 병목 현상을 최소화할 수 있습니다.
- **예외 처리**: 각 넘버링 단계(autonumber)에서 발생할 수 있는 에러 상황(DB 장애, 통신 장애 등)을 백엔드 비즈니스 로직 구현 시 참고하세요.
