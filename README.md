# Readpoint — Book Relationship Visualization Service

> 스포일러 없이 책을 읽고, 등장인물 관계를 시각화하는 RAG 기반 독서 서비스

## 📌 프로젝트 개요

Readpoint는 EPUB 기반 전자책을 읽으면서 스포일러 없이 등장인물 관계를 탐색할 수 있는 AI 독서 서비스입니다. 책을 읽는 현재 진도에 맞춰 인물 관계 그래프가 동적으로 업데이트되고, RAG 기반 AI 독서 메이트가 읽은 범위 내에서만 답변합니다. 관리자는 EPUB을 업로드하면 Azure ADF 파이프라인이 자동으로 캐릭터 추출·관계 분석·그래프 구축까지 처리합니다.

**전체 파이프라인 흐름:**
```
[메타데이터 파싱]
metadata_parser (epub 업로드 → 메타데이터 추출)
↓
[1차 파이프라인 - 캐릭터 추출 및 관계 분석]
chapter_split → openai_extract → normalize_characters → save_normalized_analysis → book_graph_refine
↓
[2차 파이프라인 - 요약 생성]
migrate_graph → generate_progress_summary
```

## 🎯 핵심 기능

- **스포일러 방지**: 현재 읽은 챕터 범위 내에서만 AI가 답변
- **인물 관계도**: 사건별로 동적 업데이트되는 등장인물 관계 그래프
- **AI 독서 메이트**: 읽은 범위 내에서만 답변하는 RAG 기반 토론 파트너
- **진도 동기화**: 마지막 읽은 위치 자동 저장 및 이어읽기

## 🏗️ 전체 아키텍처

![아키텍처](./images/3차_프로젝트-전체_아키텍처_drawio.png)

## 🗂️ 데이터 파이프라인

### 1차 파이프라인 — 캐릭터 추출 및 관계 분석
![1차 파이프라인](./images/1차_파이프라인.png)

`chapter_split` → `get_chapters` → ForEach `openai_extract` → `normalize_characters` → `save_normalized_analysis` → `book_graph_refine`

### 2차 파이프라인 — 독서 진행 요약 생성
![2차 파이프라인](./images/2차_파이프라인.png)

`migrate_graph_endpoint` → `get_progress_events` → ForEach `generate_progress_summary`

#### 파이프라인 성능 결과 (최종 적용: ForEach 병렬처리 최대 3)

| 책 제목   | 챕터 수 | 문단 수 | 전체 시간   | 비용      |
| ------ | ---- | ---- | ------- | ------- |
| 어머니와 딸 | 6    | 1861 | 11m 29s | 약 1300원 |
| 선화공주   | 5    | 744  | 8m 58s  | 약 800원  |
| 순정해협   | 7    | 1708 | 9m 51s  | 약 1400원 |
| 거리의 목가 | 12   | 663  | 11m 45s | 약 1900원 |
| 꿈      | 22   | 947  | 20m 11s | 약 3600원 |
| 흙      | 5    | 6304 | 8m 10s  | 약 1500원 |


### 파이프라인 성공 실행 (Gantt)
![성공 Gantt](./images/adf_success_gantt.png)

## 🗄️ 데이터베이스 ERD

![ERD](./images/ERD.png)

## 🖥️ 서비스 화면

### 랜딩 페이지
![랜딩](./images/front_landing.png)
![랜딩2](./images/front_landing2.png)

### 로그인
![로그인](./images/front_login.png)

### 서재
![서재](./images/front_library.png)

### EPUB 뷰어
![뷰어](./images/front_viewer.png)

### 인물 관계도
![관계도](./images/front_graph.png)

### AI 독서 메이트
![AI](./images/fron_ai.png)

### 관리자 흐름 (영상)
추후 업로드 예정

## 🔧 트러블슈팅

### Content Filter 오류 대응
![에러](./images/adf_error_contentfilter.png)
![에러 팝업](./images/adf_error_contentfilter_popup.png)

Azure OpenAI Content Management Policy로 인해 `violence: filtered: true` 오류 발생 → FILTERED 상태 분리 처리로 해결

### Too Many Requests 처리
![요청 초과](./images/adf_error_too_many_request.png)
![요청 초과 Gantt](./images/adf_error_too_many_request_gantt.png)

ForEach Batch Count 20 → 1 → 3 순차 조정으로 Rate Limit 해소

### Managed VNet Private Endpoint 구성
![IR](./images/adf_IR.png)
![Private Endpoint](./images/adf_private_endpoint.png)

ADF에서 PostgreSQL 접근 시 퍼블릭 IP 차단 이슈 → Managed VNet + Private Endpoint 구성으로 해결

## 🔄 Airflow DAG 마이그레이션 (개인 실습)

프로젝트 완료 후 ADF 파이프라인을 Apache Airflow DAG로 개인적으로 재설계했습니다.

- **환경**: Docker + Airflow 2.10.5 (로컬)
- **DAG**: `dags/readpoint_dag.py`
- **주요 변경점**:
    - ADF ForEach → Python for 루프로 구현
    - XCom으로 태스크 간 books_id 전달
    - 실패 태스크만 선택적 재실행 가능

- **레포**: [readpoint-airflow](https://github.com/bagg8234-lab/readpoint-airflow)
- **Velog 시리즈**: [ADF에서 Airflow로](https://velog.io/@rldnjs0906/series/Azure-Data-Factory-%EC%93%B0%EA%B3%A0-Airflow%EB%A1%9C-%EB%84%98%EC%96%B4%EA%B0%84-%EC%9D%B4%EC%9C%A0)

## 📁 레포지토리 구성

| 레포 | 설명 |
|------|------|
| [metadatafunction](https://github.com/3dt-3rd-project-org/metadatafunction) | 메타데이터 파싱 함수 |
| [insight-service](https://github.com/3dt-3rd-project-org/insight-service) | Application Insights 모니터링 |
| [reader-service](https://github.com/3dt-3rd-project-org/reader-service) | 도서 조회 서비스 |
| [auth-service](https://github.com/3dt-3rd-project-org/auth-service) | 인증 서비스 |
| [embedding-service](https://github.com/3dt-3rd-project-org/embedding-service) | RAG 임베딩 서비스 |
| [admin-service](https://github.com/3dt-3rd-project-org/admin-service) | 관리자 대시보드 |
| [webhook-service](https://github.com/3dt-3rd-project-org/webhook-service) | 웹훅 서비스 |
| [function](https://github.com/3dt-3rd-project-org/function) | Azure Functions 파이프라인 |
| [neo4j-data-pipeline](https://github.com/3dt-3rd-project-org/neo4j-data-pipeline) | Neo4j 그래프 DB 파이프라인 |
| [readpoint-frontend](https://github.com/3dt-3rd-project-org/readpoint-frontend) | React 기반 프론트엔드 |
| [readpoint-airflow](https://github.com/bagg8234-lab/readpoint-airflow) | Airflow DAG 마이그레이션 (개인 실습) |

## 🛠️ 기술 스택

| 분류 | 기술 |
|------|------|
| Frontend | React, Azure Static Web Apps |
| Backend | Azure Functions, Azure APIM |
| AI | GPT-5, GPT-4o-mini, text-embedding-ada-002 |
| Data | Azure ADF, pgvector, Neo4j, Logic Apps |
| Realtime | Azure Web PubSub |
| Monitoring | Azure Application Insights |
| Infra | Azure VM (Ubuntu 24.04), Docker, Managed VNet |
| Orchestration | Azure ADF, Apache Airflow |

## 👤 담당 작업

- React 프론트엔드 전체 UI/UX 개발 (랜딩, 서재, 뷰어, 관계도, AI 독서 메이트)
- Admin 대시보드 실제 API 연동 및 epub 업로드·도서 상태 파이프라인 UI 구현
- Application Insights 연동 API 개발 및 실시간 모니터링 화면 구현
- LLM 기반 AI 독서 메이트 기능 개발 (Query Rewriting, 스포일러 필터링)
- ADF 데이터 파이프라인 작업 참여 (캐릭터 추출·정규화)
- ADF 파이프라인을 Airflow DAG로 재설계 (Docker 로컬 환경 구축, XCom/ForEach 구현)
