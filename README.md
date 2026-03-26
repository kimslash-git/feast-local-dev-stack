# 프로젝트명 수정 테스트
> [English](README.en.md) | 한국어

Feast 기반 feature store를 빠르게 구축·테스트·데모할 수 있는 로컬 개발 환경입니다.

## 개요

`feast-local-dev-stack`은 샘플 데이터, materialization, online 조회가 포함된 소규모 Feast 로컬 개발 환경을 구성하는 스타터 저장소입니다. 몇 분 안에 feature store를 처음부터 실행 가능한 상태까지 구성할 수 있습니다.

이 프로젝트는 프로덕션 배포보다 **로컬 개발과 데모**에 집중합니다.

## 사전 요구사항

- [Docker](https://docs.docker.com/get-docker/) 및 [Docker Compose](https://docs.docker.com/compose/install/)
- [Make](https://www.gnu.org/software/make/) (macOS/Linux에는 보통 기본 설치됨)
- 로컬 Python 설치 불필요—모든 작업은 컨테이너 내부에서 실행됨

## 구성 요소

| 구성 요소 | 설명 |
|----------|------|
| Feast | v0.61.0, Redis online store 포함 |
| Redis | 7-alpine, 포트 6379 노출 |
| Feast Web UI | 포트 8888 베타 UI (`make ui` 실행) |
| Jupyter Lab | 포트 8889, 단계별 워크스루 노트북 (`make notebook` 실행, `notebooks/feast_walkthrough.ipynb`) |
| 예시 feature repo | 이커머스 고객(customer_stats) feature view, 합성 데이터 생성기 |
| 데모 앱 | Online feature 조회 (`app/fetch_demo.py`), Historical features (`app/training_dataset_demo.py`) |
| Makefile 타깃 | `up`, `apply`, `materialize`, `fetch`, `ui`, `notebook`, `bootstrap`, `shell` |

## 빠른 시작

### 1. 클론 및 준비

```bash
git clone https://github.com/VectorBlocks/feast-local-dev-stack.git
cd feast-local-dev-stack
```

### 2. 서비스 시작

```bash
make up
```

`feast-dev` 컨테이너를 빌드하고 Redis를 시작합니다. 서비스 준비까지 몇 초 기다리세요.

### 3. 피처 정의 적용

```bash
make apply
```

Feast에 feature repository(entities, feature views)를 등록합니다.

### 4. 피처 Materialize

```bash
make materialize
```

합성 데이터를 생성하고, `feast materialize-incremental`을 실행해 Redis에 피처를 기록합니다.

### 5. Online 피처 조회

```bash
make fetch
```

샘플 customer ID에 대한 online 피처를 조회하는 데모를 실행합니다. 피처 값이 담긴 딕셔너리가 출력됩니다.

### 6. (선택) Feast Web UI로 탐색

```bash
make ui
```

[http://localhost:8888](http://localhost:8888)에서 Feast Web UI를 엽니다. feature views, entities 및 관계를 탐색할 수 있습니다. `make apply` 이후에 실행하세요.

### 7. (선택) Jupyter 노트북 워크스루

```bash
make notebook
```

[http://localhost:8889](http://localhost:8889)에서 Jupyter Lab을 실행합니다. `notebooks/feast_walkthrough.ipynb`에서 단계별 피처스토어 워크스루를 실행할 수 있으며, 접속 시 워크스루 노트북이 기본으로 열립니다. `make apply`, `make materialize` 완료 후 실행하세요.

## 아키텍처

```text
합성 데이터 (Parquet)
        ↓
피처 정의 (example_repo/)
        ↓
feast apply → Registry (SQLite)
        ↓
feast materialize-incremental
        ↓
Redis Online Store
        ↓
Online 피처 조회 (app/fetch_demo.py)
```

## 피처 정의

이 프로젝트에 포함된 예시 피처 구조입니다. `make fetch` 또는 `make materialize` 실행 시 다루게 되는 데이터를 이해하는 데 도움이 됩니다.

### Entity

| Entity | Join key | 설명 |
|--------|----------|------|
| `customer` | `customer_id` | 고객 단위로 피처 조회 (예: 1001, 1002, 1003) |

### Feature view: `customer_stats`

`customer_stats`는 이커머스 고객별 구매 관련 피처를 담은 Feature View입니다.

| Feature name | 타입 | 설명 | 예시 값 |
|--------------|------|------|---------|
| `purchase_count_30d` | Int64 | 최근 30일 구매 횟수 | 0~15 |
| `avg_order_value_30d` | Float32 | 최근 30일 평균 주문 금액 | 0~150,000 |
| `days_since_last_purchase` | Int64 | 마지막 구매 후 경과 일수 | 0~30 |

### 샘플 데이터

- **3명의 고객**: `customer_id` 1001, 1002, 1003
- **기간**: 고정 10일 (2026-03-01 ~ 2026-03-10), 고객당 10일치 데이터 모두 존재
- **생성**: `example_repo/generate_sample_data.py` (고정 데이터, Online/Historical 차이 확인용)
- **저장 위치**: `example_repo/data/customer_stats.parquet` (런타임 생성, `.gitignore` 대상)

### 조회 예시

```python
# Online: customer_id로 최신 피처 조회
store.get_online_features(
    features=["customer_stats:purchase_count_30d", "customer_stats:avg_order_value_30d", "customer_stats:days_since_last_purchase"],
    entity_rows=[{"customer_id": 1001}],
)
```

## Makefile 참조

| 타깃 | 설명 |
|------|------|
| `make up` | Docker Compose 시작 (Redis + feast-dev) |
| `make down` | 컨테이너 중지 및 제거 |
| `make shell` | feast-dev 컨테이너에서 bash 셸 실행 |
| `make apply` | `feast apply`로 feature repo 적용 |
| `make materialize` | 데이터 생성 + `feast materialize-incremental` |
| `make fetch` | Online 피처 조회 데모 실행 |
| `make ui` | Feast Web UI 시작 (http://localhost:8888) |
| `make notebook` | Jupyter Lab 시작 (http://localhost:8889), 워크스루 노트북 제공 |
| `make bootstrap` | 데이터 생성 + apply 한 번에 실행 |

## 저장소 구조

```text
.
├── docker-compose.yml      # Redis + feast-dev 서비스
├── Dockerfile              # Feast 0.61.0, pandas, pyarrow
├── Makefile                # 편의 타깃
├── .env.example            # 환경 변수 템플릿
├── example_repo/           # Feast feature repository
│   ├── feature_store.yaml  # 프로젝트 설정, Redis 연결
│   ├── entities.py         # Customer entity
│   ├── feature_views.py    # customer_stats feature view
│   ├── generate_sample_data.py
│   └── data/               # Parquet 파일 (런타임 생성)
├── app/
│   ├── fetch_demo.py       # Online 피처 조회
│   └── training_dataset_demo.py  # Historical 피처
├── scripts/                # Bash 스크립트
├── docs/
└── notebooks/              # Jupyter 워크스루 (feast_walkthrough.ipynb)
```

## 문제 해결

| 증상 | 해결 방법 |
|------|-----------|
| Redis `Connection refused` | `make up`이 완료되었고 Redis가 실행 중인지 확인. `docker compose ps`로 확인. |
| `feast apply` 실패 | `feast-dev` 컨테이너에 접근 가능한 환경인지 확인. `make apply` 사용(컨테이너 내부에서 실행됨). |
| `make fetch`에서 피처가 반환되지 않음 | `make materialize`를 먼저 실행. Online 조회 전에 피처 materialize 필요. |
| 포트 6379 사용 중 | `docker-compose.yml`에서 Redis 포트 변경 후 `feature_store.yaml`도 맞춰 수정. |

### 스키마 변경 시 취해야 할 조치

Entity나 Feature View 등 스키마를 변경한 경우, 기존 registry와 Redis 데이터를 초기화해야 합니다.

```bash
make down
rm -rf example_repo/data/
make up
make apply
make materialize
make fetch
```

- `example_repo/data/`에 registry(registry.db)와 Parquet 파일이 있으며, 스키마 변경 시 불일치가 발생할 수 있습니다.
- Redis는 `make down -v`로 초기화되며, `example_repo/data/`는 호스트에 남아 있으므로 수동 삭제가 필요합니다.

## 참고 사항

- 샘플 데이터와 피처 정의는 의도적으로 일반적인 형태로 구성되어 있습니다.
- 이 저장소는 오픈소스 스타터, 데모 스택, 내부 부트스트랩 템플릿으로 활용하기 적합합니다.
- 실제 워크로드에 적용하기 전에는 예시 피처 정의와 데이터 소스를 교체하세요.

## 로드맵

- [x] 최소한의 로컬 개발 스캐폴드
- [x] Jupyter 노트북 워크스루
- [x] 추가 샘플 feature views (1~2개, 학습용)

## 라이선스

Apache License 2.0
