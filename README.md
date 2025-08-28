# go-project-structure
go 프로젝트 구조 학습 정리

## 참고
- https://go.dev/doc/modules/layout
- https://github.com/golang-standards/project-layout/blob/master/README_ko.md
- https://www.alexedwards.net/blog/11-tips-for-structuring-your-go-projects
- https://medium.com/@janishar.ali/how-to-architecture-good-go-backend-rest-api-services-14cc4730c05b
- https://medium.com/geekculture/how-to-structure-your-project-in-golang-the-backend-developers-guide-31be05c6fdd9

## internal 디렉터리
외부에서 임포트(import)되지 않도록 강제 보호하는 용도. (내부 전용)
- internal 디렉터리에 넣는 것들
  - 외부에 노출하고 싶지 않는 내부 로직
    - 예: DB 접근 레이어, 캐시 처리, 내부 유틸
    - internal/db, internal/cache, internal/config
  - 공용 API가 아닌, 특정 프로젝트 내부 전용 기능
    - 예: 서버 초기화 로직, 미들웨어
    - internal/server, internal/middleware
  - 라이브러리화할 필요 없는 코드
    - 예: 특정 프로젝트에 종속된 구현체
    - internal/serializer, internal/auth
  - 넣지 말아야 할 것들
    - 다른 프로젝트나 모듈에서 재사용할 수 있는 일반 라이브러리성 코드
      - pkg 디렉터리에 둔다

## 웹 기반 구조를 통한 예시
```
mygame/
├── cmd/gameserver/main.go
├── internal/
│ ├── api/
│ │ ├── http/ # REST
│ │ └── ws/ # WebSocket
│ ├── app/ # 게임 도메인 로직 (lobby, room, battle)
│ ├── service/ # 비즈니스 흐름 제어
│ ├── repository/ # DB/Redis CRUD
│ ├── infra/ # DB/Redis/MQ 연결
│ └── domain/ # 엔터티 정의
└── go.mod
```

- 흐름
  - cmd/server/main.go
    - 진입점
    - DB 연결 초기화 (internal/db)
    - 서비스/핸들러
    - chi.Router로 라우팅 설정 후 서버 실행
  - interanl/api (Handler)
    - HTTP 핸들러 (chi 라우팅)
    - HTTP 요청 파싱 → service 호출 → 응답 작성
  - internal/service
    - 비즈니스 로직 담당
    - DB 접근이 필요하면 repository 호출
  - internal/repository
    - MySQL CRUD 담당 (database/sql 또는 gorm 등 ORM 사용 가능)


- handler와 service

|구분|Handler|Service|
|---|--------|-------|
|위치|API 레이어|비즈니스 로직 레이어|
|입력|HTTP Request|순수 데이터(struct, 값)|
|출력|HTTP Response|비즈니스 결과(struct, error)|
|의존성|웹 프레임워크(chi 등)|Repository(DB), 다른 Service|
|테스트|통합 테스트 위주|단위 테스트 쉬움|

  - 분리하는 이유
    - 관심사 분리
      - Handler는 네트워크 입출력 담당
      - Service는 비즈니스 로직 담당
    - 재사용성
      - Service는 CLI, gRPC 등 다른 인터페이스에서도 재사용 가능
    - 테스트 용이성
      - Service는 HTTP 없이도 단위 테스트 가능
 - 정리
   - Handler
      - HTTP 요청/응답 처리 담당자.
      - 요청(Request)를 읽고 Service 호출 그리고 결과(Response)로 변환
   - Service
     - “무엇을 할지”를 정의하는 비즈니스 로직 담당자
     - 실제 업무 규칙(비즈니스 로직) 처리
     - DB나 외부 시스템에 접근해야 할 경우 Repository 호출


## 게임 서버 구조를 통한 규모 있는 프로젝트 예시
```
mygame/
 ├── cmd/
 │    └── gameserver/           # main.go (엔트리포인트)
 │
 ├── internal/
 │    ├── api/                  # 외부 인터페이스
 │    │    ├── http/            # REST API (로그인, 회원가입 등)
 │    │    └── ws/              # WebSocket/GRPC (게임 통신)
 │    │
 │    ├── app/                  
 │    │    └── application.go  (조립/런처)
 │    │
 │    ├── service/              # 도메인 서비스 (비즈니스 규칙)
 │    │    ├── auth/
 │    │    │    └── auth_service.go  # 계정/인증 도메인
 │    │    ├── match/
 │    │    │    └── match_service.go # 로비/매치메이킹
 │    │    └── room/
 │    │         └── room_service.go  # 룸/세션 관리
 │    │
 │    ├── repository/           # 데이터 저장소 접근 (DB/Redis 등)
 │    │    ├── user_repo.go
 │    │    ├── room_repo.go
 │    │    └── cache_repo.go
 │    │
 │    ├── infra/                # 인프라 (외부 시스템 연동)
 │    │    ├── db/              # MySQL 연결
 │    │    ├── redis/           # Redis 연결
 │    │    └── mq/              # 메시지 큐(Kafka, NATS 등)
 │    │
 │    ├── server/               # 서버 실행, 라우팅 설정
 │    │    ├── http_server.go
 │    │    ├── ws_server.go
 │    │    └── grpc_server.go
 │    │
 │    └── domain/               # 공용 엔터티/도메인 모델
 │         ├── user.go
 │         ├── room.go
 │         └── match.go
 │
 ├── pkg/                       # 재사용 가능한 범용 유틸 (필요할 때만)
 │    ├── logger/
 │    ├── config/
 │    └── errors/
 │
 └── go.mod
```

- 계층별 역할
    - api/
        - 네트워크 입출력 담당 (Controller/Handler)
        - REST → 회원가입, 로그인
        - WebSocket/gRPC → 실시간 게임 패킷 처리
    - app/
        - 서버 실행, 라우팅, 초기화 같은 것을 담당.
    - service/
        - 비즈니스 규칙/흐름 제어
        - 예: "로그인 시 토큰 발급 + Redis에 세션 등록"
    - repository/
        - DB, Redis 같은 저장소 접근
        - 순수하게 CRUD 담당
    - infra/
        - 외부 시스템 연동 (DB, Redis, MQ, 외부 API)
    - domain/
        - 엔터티(Entity), 값 객체(Value Object) 정의
        - 예: `User`, `Room`, `Match`
     
- 위와 같이 나눈 이유
    1. **도메인 중심**: 게임은 "계정/로비/룸/배틀" 같은 **도메인 단위로 나누는 게 자연스러움**
    2. **실시간 처리 분리**: REST API와 WebSocket 같은 실시간 처리를 분리하면 유지보수 편리
    3. **확장성**: DB, Redis, MQ 같은 인프라 계층을 분리해 두면, 나중에 교체하거나 확장하기 쉬움
    4. **테스트 용이성**: Service, App, Repository를 분리하면 각각 단위 테스트가 쉬움
