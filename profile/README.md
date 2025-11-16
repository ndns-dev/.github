# NDNS (내돈내산) Project

NDNS는 네이버 블로그 검색 결과에 포함된 수많은 포스트 중에서, 사용자가 광고나 협찬이 아닌 순수한 정보성 콘텐츠를 찾을 수 있도록 돕는 시스템입니다. 다단계 분석 파이프라인을 통해 포스트의 광고성 여부를 판별하고, 실시간에 가까운 협찬 여부 분석 결과를 제공하는 것을 목표로 합니다.

👉 **서비스 바로가기**: [https://www.ndns.site](https://ndns.site)

## 🚀 주요 기능 (Key Features)

-   **실시간 분석**: 네이버 검색 API를 통해 얻은 블로그 포스트를 실시간으로 분석합니다.
-   **다단계 탐지 로직**: 포스트의 텍스트, HTML 구조, 이미지 등 다양한 요소를 종합하여 정확하게 광고성 콘텐츠를 판별합니다.
-   **비동기 업데이트**: 초기 분석 후 이미지 OCR과 같이 시간이 소요되는 작업은 비동기적으로 처리되며, Server-Sent Events(SSE)를 통해 클라이언트에 점진적으로 업데이트된 결과를 전송합니다.
-   **확장 가능한 아키텍처**: Go 기반의 API 서버와 AWS Lambda, SQS, DynamoDB 등 클라우드 네이티브 기술을 활용하여 트래픽 변화에 유연하게 대응할 수 있도록 설계되었습니다.

## ⚙️ 전체 시스템 아키텍처 (Overall System Architecture)

<details>
<summary>클릭하여 전체 시스템 아키텍처 보기</summary>

```mermaid
sequenceDiagram
    autonumber
    participant 클라이언트
    participant 라우터
    participant API 서버
    participant 네이버 API
    participant DynamoDB
    participant OCR Lambda
    participant SQS

    클라이언트->>라우터: 1. /api/v1/search (검색어)
    라우터->>API 서버: 요청 프록시
    API 서버->>네이버 API: 네이버 검색 API 호출
    네이버 API-->>API 서버: 검색결과 (10개 포스트)

    API 서버->>DynamoDB: 기존 분석 결과 조회
    DynamoDB-->>API 서버: 기존 분석 결과 반환

    note over API 서버: 1차 분석 프로세스 실행 (자세한 내용은 하단 참조)
    API 서버->>SQS: 이미지 OCR 필요 시 SQS에 작업 전송

    API 서버-->>라우터: 1차 분석 결과 (협찬 여부+대기 상태), 헤더: X-SSE-Token, X-Req-Id
    라우터-->>클라이언트: 응답 전송

    클라이언트->>라우터: SSE 연결 요청 (/stream?sseId=ReqId, with SSE-Token)
    note over 클라이언트, 라우터: SSE 연결 완료

    note over SQS, OCR Lambda: 비동기 OCR 프로세스 실행 (자세한 내용은 하단 참조)
    OCR Lambda->>라우터: OCR 분석 결과 전달 (요청 ID 포함)
    라우터->>클라이언트: SSE 메시지 전송 (요청 ID 채널)
```

</details>

## 🔬 1차 분석 프로세스 (Initial Analysis Process)

<details>
<summary>클릭하여 1차 분석 프로세스 보기</summary>

```mermaid
graph TD
    A[시작: 새 포스트 분석] --> B{설명 확인};
    B -- "협찬 문구 발견" --> C[협찬으로 표시];
    B -- "협찬 문구 없음" --> D[포스트 본문 전체 크롤링];
    D --> E{작성일 >= 2025년?};
    E -- "예" --> F[첫 문단 분석];
    E -- "아니오" --> G[첫 문단 + 마지막 문단 분석];
    F --> H{협찬 문구 발견?};
    G --> H;
    H -- "예" --> C;
    H -- "아니오" --> I[이미지 도메인 분석];
    I -- "협찬 도메인 발견" --> C;
    I -- "협찬 도메인 없음" --> J[비동기 OCR 대기열에 추가];
    J --> K[분석 대기로 표시];
```

</details>

## 🖼️ OCR 아키텍처 (OCR Architecture)

<details>
<summary>클릭하여 OCR 아키텍처 보기</summary>

```mermaid
graph TD
    subgraph "OCR 처리"
        A[SQS] -- "OCR 작업 수신" --> B(OCR Lambda - Go);
        B -- "Tesseract 데이터 가져오기" --> S3[(S3 버킷)];
        B -- "이미지 URL 로드" --> Image[이미지];
        Image --> Size_Check{이미지 크기 확인};
        Size_Check -- "임계값 초과" --> Resize[이미지 리사이징];
        Size_Check -- "임계값 이하" --> Perform_OCR[OCR 수행];
        Resize --> Perform_OCR;
        Perform_OCR -- "OCR 결과" --> B;
        B -- "POST /api/v1/search/analyze/cycle" --> 라우터;
        라우터 -- "API 서버로 프록시" --> API_서버[API 서버];
    end
```

</details>

## 🌐 라우팅 및 헬스체크 아키텍처 (Routing & Health Check Architecture)

<details>
<summary>클릭하여 라우팅 및 헬스체크 아키텍처 보기</summary>

```mermaid
graph TD
    subgraph "사용자 요청 처리"
        클라이언트 -- "API 요청" --> 라우터;
        라우터 -- "화이트리스트 확인" --> 라우터_결정{요청 허용?};
        라우터_결정 -- "예" --> 프록시;
        라우터_결정 -- "아니오 (허용 목록에 없음)" --> 요청_거부[/"404 Not Found"/];
        subgraph "VPC"
            subgraph "API 서버 보안 그룹 (라우터 & Lambda만 허용)"
                direction LR
                API_서버_1[API 서버 1];
                API_서버_2[API 서버 2];
                API_서버_N[API 서버 N];
            end
        end
        프록시 -- "최적 서버로 프록시" --> API_서버_2;
    end

    subgraph "주기적인 메트릭 수집 (1분마다)"
        EventBridge[AWS EventBridge] -- "트리거" --> 메트릭_Lambda(메트릭 Lambda - Python);
        메트릭_Lambda -- "GET /metrics" --> API_서버_1;
        메트릭_Lambda -- "GET /metrics" --> API_서버_2;
        메트릭_Lambda -- "GET /metrics" --> API_서버_N;
        API_서버_1 --> 메트릭_Lambda;
        API_서버_2 --> 메트릭_Lambda;
        API_서버_N --> 메트릭_Lambda;
        메트릭_Lambda -- "메트릭 전송" --> 프로메테우스_서버[프로메테우스 서버];
        메트릭_Lambda -- "서버 상태 업데이트" --> 라우터;
    end

    style 라우터 fill:#f9f,stroke:#333,stroke-width:2px
```

</details>

### 보안 강화 (Security Enhancements)

1.  **라우터 레벨 접근 제어 (Nginx Whitelist)**
    *   `Router` 서버(Nginx)는 사전에 정의된 API 엔드포인트(e.g., `/api/v1/search`)에 대한 요청만 허용하는 **화이트리스트** 기반으로 동작합니다.
    *   화이트리스트에 존재하지 않는 모든 경로로의 요청은 `404 Not Found`로 처리되어, 불필요한 내부 시스템 접근을 원천적으로 차단합니다.

2.  **네트워크 레벨 접근 제어 (AWS Security Group)**
    *   각 `API Server`는 **라우터 서버**와 **AWS Lambda**의 IP 주소에서 오는 트래픽만 허용하도록 AWS 보안 그룹(Security Group)이 설정되어 있습니다.
    *   이를 통해 라우터를 우회하여 API 서버에 직접 접근하는 것을 막아 시스템의 보안을 강화합니다.

## 🛠️ 기술 스택 (Tech Stack)

-   **Backend**: Go
-   **Automation (Lambda)**: Go, Python
-   **API**: REST, Server-Sent Events (SSE)
-   **Cloud Services**: AWS Lambda, SQS, DynamoDB, S3, EventBridge
-   **Infrastructure**: Docker, Nginx
-   **CI/CD**: GitHub Actions
