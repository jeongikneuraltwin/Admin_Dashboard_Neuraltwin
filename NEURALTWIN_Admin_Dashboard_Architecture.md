
# NEURALTWIN 관리자 콘솔 통합 아키텍처

- **버전**: v11
- **상태**: Draft (내부 설계용)
- **대상 시스템**: NEURALTWIN Admin Web (운영·관리자용 콘솔)

---

## 0. 문서 개요

### 0.1 목적

본 문서는 NEURALTWIN 플랫폼의 **관리자용 웹 콘솔(Admin Console)** 아키텍처를 정의한다.
고객(테넌트)이 사용하는 분석·시뮬레이션·3D 디지털 트윈 UI와는 별도의 채널로, 다음 목적을 가진다.

- 테넌트/사용자/플랜/사용량 관리
- 데이터 파이프라인 및 품질 모니터링
- Ontology & Graph 스키마 및 인스턴스 거버넌스
- 3D Digital Twin 제작·배포 파이프라인 관리
- AI/시뮬레이션 상태 및 오류 모니터링
- 시스템/에러/성능 모니터링
- 운영자가 고객 요청·문제를 효율적으로 처리할 수 있는 프로세스 제공

### 0.2 범위

- **포함**
  - Admin 전용 웹 UI 정보 구조(IA)
  - 주요 페이지 및 노출 데이터 정의
  - 운영 플로우 및 대응 시나리오
  - 권한/보안 개요
  - 관리자용 메타 데이터 구조(고수준)

- **제외**
  - 고객용(테넌트) 애플리케이션 상세 아키텍처
  - 개별 컴포넌트 수준 UI/UX 상세 디자인
  - 인프라/네트워크 레벨 배포 구성 세부사항

---

### 0.3 상위 채널 아키텍처 맥락

NEURALTWIN 전체 아키텍처는 **하나의 코어 플랫폼(데이터·시뮬레이션·라이선스)** 을
다음 **3개의 채널**이 공유하는 구조이다.

```text
[1. Public Website]       ┐
[2. Customer Dashboard]   ┤  →  [API Gateway / BFF]
[3. HQ Admin Dashboard]   ┘

        ↓ (공통으로 사용)

[Identity & Billing Service]      (회원·조직·권한·구독)
[Tenant / License Service]        (브랜드·매장·라이선스)
[Data Platform (NEURALMIND)]      (POS/CRM/센서/외부데이터 통합·온톨로지)
[Simulation & AI (NEURALTWIN)]    (레이아웃/수요/재고/가격/프로모션 시뮬레이션)
[Sensor Ingestion (NEURALSENSE)]  (와이파이·디바이스 수집)
[Storage]                         (Postgres, Data Warehouse, Object Storage)
[Observability / Logs]            (모니터링·알람)
```

각 채널의 역할은 다음과 같다.

- **Public Website**
  - 가입/로그인, 가격/플랜 안내, 구독 결제(Stripe/PG) 등
  - 가입/결제 이후 조직(Brand) + 기본 라이선스를 생성하는 **입구 역할**
- **Customer Dashboard (웹 대시보드)**  
  - 브랜드/HQ/매장 팀이 실제로 쓰는 운영 도구
  - 매장/매출/방문/센서 데이터를 기반으로 분석·시뮬레이션·디지털 트윈을 실행
- **HQ Admin Dashboard (본 문서의 대상)**  
  - NEURALTWIN 팀 내부 전용 콘솔
  - 전 테넌트/라이선스/온톨로지/데이터 파이프라인/AI 모델/3D 제작 파이프라인/비용 등을
    중앙에서 모니터링·조정하는 **컨트롤 타워 역할**

본 문서에서 정의하는 **관리자 콘솔(= HQ Admin Dashboard)** 은 위 3번째 채널에 해당하며,
Website / Customer Dashboard 와 **동일한 Auth·Org·License·Data/Simulation Core** 를 공유하지만
노출되는 화면/기능과 권한 모델이 완전히 다르다.


---

## 1. 관리자 콘솔의 역할 및 페르소나

### 1.1 주요 역할

관리자 콘솔은 다음 역할을 수행한다.

1. **비즈니스 운영 관리**
   - 테넌트(고객사)·플랜·사용량·라이선스 관리
   - CS/영업 관점에서의 고객 상태 파악 및 조치

2. **데이터 품질 및 파이프라인 관리**
   - 임포트/동기화/집계 파이프라인 모니터링
   - 데이터 건강도(Data Health) 관리 및 재처리

3. **Ontology & Graph 거버넌스**
   - 온톨로지(Entity/Relation 타입) 정의 및 변경
   - Graph 인스턴스(Entities & Relations) 모니터링 및 품질 관리
   - **Graph는 관리자 전용**이며, 고객 UI에서는 직접 노출되지 않음

4. **3D Digital Twin 제작·배포**
   - 고객이 자체 3D 모델이 없는 경우,
     내부/외주 팀을 통한 3D 모델링·텍스처링·베이크·glb 추출 및 메타데이터 생성 관리
   - 완성된 SceneRecipe 및 3D Assets 를 고객 테넌트로 배포

5. **AI & Simulation/Recommendation 모니터링**
   - 각 테넌트의 시뮬레이션 실행 현황, 실패율, 응답 품질 모니터링
   - 이상 패턴(특정 테넌트의 비정상적인 실패 증가 등) 탐지

6. **시스템 안정성 및 보안 관리**
   - Edge Function/API 에러 모니터링
   - 기본 성능 트렌드 모니터링
   - RLS/권한 오작동 감시

### 1.2 내부 페르소나

- **운영/CS 담당자**
  - 테넌트 및 사용자 현황 파악
  - 고객 문의에 대한 상황 확인 및 간단한 조치 수행

- **데이터/AI 담당자**
  - 데이터 파이프라인 상태 및 품질 관리
  - Ontology/Graph 스키마 및 인스턴스 품질 점검
  - AI/시뮬레이션 이상 패턴 분석

- **시스템/플랫폼 엔지니어**
  - Edge Functions 및 주요 API 상태 모니터링
  - 성능 이슈 및 에러 핫스팟 파악
  - 운영 자동화 도입

- **3D 콘텐츠/디지털 트윈 담당자**
  - 매장 3D 모델링 제작 요청 관리
  - 3D Asset/Scene 품질 검수 및 배포

---

## 2. 전체 IA 구조 (HQ Admin 콘솔 메뉴)

### 2.1 상위 IA 개요

HQ Admin 콘솔은 다음 **4개의 상위 도메인 그룹**과 하위 기능 단위 섹션으로 구성된다.

```text
NEURALTWIN HQ ADMIN CONSOLE

1. Tenants / Billing
   ├─ Tenants                (/hq/tenants)
   ├─ Licenses               (/hq/licenses)
   ├─ Subscriptions          (/hq/subscriptions)
   └─ Usage                  (/hq/usage)

2. Data & Ontology
   ├─ Ontology Schema        (/hq/ontology-schema)
   ├─ Data Sources           (/hq/data-sources)
   ├─ Pipelines              (/hq/pipelines)
   └─ 3D Digital Twin Management
        ├─ 제작 요청 관리    (/hq/3d-requests)
        └─ 3D 프로덕션 관리  (/hq/3d-studio)

3. Simulation & Models
   ├─ Simulation Config      (/hq/simulation-config)
   ├─ Model Versions         (/hq/model-versions)
   ├─ Experiments            (/hq/experiments)
   └─ AI & Simulation Monitoring
        ├─ Overview          (/hq/ai-sim)
        └─ Tenant Detail     (/hq/ai-sim/:tenantId)

4. Ops / Security
   ├─ Users                  (/hq/users)
   ├─ Audit Logs             (/hq/audit-logs)
   ├─ Alerts                 (/hq/alerts)
   ├─ System & Error Console (/hq/system/errors, /hq/system/functions)
   └─ Admin Tools & Settings (/hq/tools, /hq/settings)
```

- 상위 그룹: Tenants / Billing, Data & Ontology, Simulation & Models, Ops / Security
- 하위 섹션: 실제 라우트 기준의 화면 단위로 구성된다.
- 라우팅(URL) 구조는 기능 단위 하위 섹션 기준으로 정의하고, UI 상에서 상위 그룹별로 네비게이션을 묶는다.

### 2.2 라우팅 구조 예시

```tsx
<Routes>
  {/* Dashboard (공통 HQ Admin 홈) */}
  <Route path="/hq" element={<HqHome />} />

  {/* 1. Tenants / Billing */}
  <Route path="/hq/tenants" element={<TenantListPage />} />
  <Route path="/hq/tenants/:tenantId" element={<TenantDetailPage />} />
  <Route path="/hq/licenses" element={<LicenseListPage />} />
  <Route path="/hq/subscriptions" element={<SubscriptionListPage />} />
  <Route path="/hq/usage" element={<UsageOverviewPage />} />
  <Route path="/hq/usage/:tenantId" element={<TenantUsageDetailPage />} />

  {/* 2. Data & Ontology */}
  <Route path="/hq/ontology-schema" element={<OntologySchemaPage />} />
  <Route path="/hq/data-sources" element={<DataSourcesPage />} />
  <Route path="/hq/pipelines" element={<PipelinesOverviewPage />} />
  {/* 3D Digital Twin Management (HQ 관점) */}
  <Route path="/hq/3d-requests" element={<DigitalTwinRequestListPage />} />
  <Route path="/hq/3d-studio" element={<DigitalTwinStudioPage />} />

  {/* 3. Simulation & Models */}
  <Route path="/hq/simulation-config" element={<SimulationConfigPage />} />
  <Route path="/hq/model-versions" element={<ModelVersionsPage />} />
  <Route path="/hq/experiments" element={<ExperimentsPage />} />
  {/* AI & Simulation Monitoring */}
  <Route path="/hq/ai-sim" element={<AISimOverviewPage />} />
  <Route path="/hq/ai-sim/:tenantId" element={<TenantAISimDetailPage />} />

  {/* 4. Ops / Security */}
  <Route path="/hq/users" element={<HqUsersPage />} />
  <Route path="/hq/audit-logs" element={<AuditLogsPage />} />
  <Route path="/hq/alerts" element={<AlertsPage />} />
  {/* System & Error Console */}
  <Route path="/hq/system/errors" element={<ErrorConsolePage />} />
  <Route path="/hq/system/functions" element={<EdgeFunctionsStatusPage />} />
  {/* Admin Tools & Settings */}
  <Route path="/hq/tools" element={<AdminToolsPage />} />
  <Route path="/hq/settings" element={<AdminSettingsPage />} />
</Routes>
```

---
---

## 3. 섹션별 상세 설계

> 각 섹션은 공통적으로
> - 목적
> - 화면 구조
> - 주요 화면(목록/상세 등)의 노출 데이터/정렬/관리 액션
> 구조로 정리한다.

---

### 3.1 사용자 관리

#### 3.1.1 목적

- 테넌트(조직), 매장, 사용자 계정, 권한 및 플랜 정보를 중앙에서 관리한다.
- 특정 테넌트에서 발생하는 문제를 탐색하기 위한 Entry Point 역할을 한다.

#### 3.1.2 화면 구조

1. **Tenant List**
   - 라우트: `/hq/tenants`
   - 전체 테넌트 현황을 요약해서 보여주는 목록 화면.

2. **Tenant Detail**
   - 라우트: `/hq/tenants/:tenantId`
   - 단일 테넌트의 매장/사용자/플랜/최근 활동을 통합해서 보여주는 상세 화면.

#### 3.1.3 Tenant List 뷰

**노출 데이터**

- tenant_id
- 조직명, 브랜드명
- 플랜 타입 (Free/Pro/Enterprise)
- 상태(Trial/Active/Suspended)
- 매장 개수, 활성 사용자 수
- 최근 활동 일시(마지막 로그인/마지막 데이터 업로드 등)
- Data Health Score (요약 지표, 3.3 데이터 관리와 연동)

**정렬/검색**

- 최근 활동 기준 정렬 (최신 활동 순/오래된 순)
- 플랜 타입, 상태 컬럼으로 정렬
- 브랜드/조직명 검색

**관리 액션 (목록에서의 단축 액션)**

- 테넌트 상태 변경 (Active ↔ Suspended)
- 상세 화면으로 이동
- 테넌트별 3D 제작 요청 목록 바로가기 (제작 요청 관리로 이동, tenantId 프리필)
- 테넌트별 데이터 관리 상세 바로가기

#### 3.1.4 Tenant Detail 뷰

**노출 데이터**

1. **기본 정보 패널**
   - 조직명, 브랜드명
   - 국가/타임존, 통화
   - 플랜 정보 (타입, 시작/종료일 등)
   - 라이선스 한도 (`license_management`)

2. **매장 정보 패널**
   - 매장 리스트 (`stores`, `hq_store_master`, `store_mappings`)
   - 매장별 상태(오픈/휴점), 개장일, 면적 등 기본 메타데이터

3. **사용자 계정 패널**
   - 이메일
   - 역할(Role) – admin, manager, viewer 등
   - 상태(Active/Disabled)
   - 마지막 접속일
   - (선택) SSO/IDP 연동 정보

4. **최근 활동 타임라인**
   - 데이터 임포트(성공/실패)
   - 시뮬레이션 실행 기록
   - 3D 씬 배포 이벤트
   - 주요 설정 변경(플랜 변경, 기능 플래그 변경 등)

**관리 액션**

- 테넌트 상태 변경 (예: Suspended 설정)
- 플랜/라이선스 변경
- 사용자 계정 잠금/해제
- 비밀번호 초기화 링크 발송
- 테넌트 단위 3D 디지털 트윈 요청 현황 확인
  - 해당 테넌트와 연관된 3D 제작 요청 목록 조회 (제작 요청 관리로 이동)
- 데이터 재집계 관련 액션 (핵심 링크 역할)
  - 해당 테넌트에 대한 재집계 Job 실행 이력 조회
  - 3.3 데이터 관리 또는 3.8 관리자 도구로 이동하여
    특정 스토어/기간에 대한 재집계 실행

---

### 3.2 사용량 및 결제 관리

#### 3.2.1 목적

- 테넌트별/전체 사용량을 모니터링하고, 기능·플랜 설계 및 향후 과금 정책 수립에 참고할 수 있는 데이터를 제공한다.
- 현 단계에서는 한도 설정·강제 차단 등 제한 기능은 제공하지 않고,
  사용량 추세를 파악하기 위한 관찰·리포트 용도에 초점을 둔다.

#### 3.2.2 화면 구조

1. **Usage Overview (전역 뷰)**
   - 라우트: `/hq/usage`
   - 전체 시스템 차원의 사용량/트래픽/시뮬레이션 사용 현황 요약.

2. **Tenant Usage Detail (테넌트별 상세)**
   - 라우트: `/hq/usage/:tenantId`
   - 단일 테넌트에 대한 기간별 사용량/추세/이상 패턴 확인.

#### 3.2.3 Usage Overview 뷰

**노출 데이터**

- 전역 사용량 요약
  - 전체 API 호출량 (AI, Edge Functions, 외부 API 프록시 등)
  - 스토리지 사용량 (store-data, 3D assets, logs)
  - 시뮬레이션 호출 횟수 및 성공률
  - 대시보드 조회/집계 호출량 (대략적인 사용 강도 지표)

- 플랜/세그먼트별 요약
  - 플랜 타입별 평균/총 사용량
  - 상위 N개 테넌트(사용량 기준) 리스트

**정렬/검색**

- 기간별 집계 그래프 (일/주/월 단위)
- 상위 N개 테넌트 리스트 정렬 (호출량/스토리지 사용량 기준)

**관리 액션**

- 특정 테넌트 클릭 시 Tenant Usage Detail로 이동
- 요약 리포트(전역) 다운로드 (CSV/PDF)

#### 3.2.4 Tenant Usage Detail 뷰

**노출 데이터**

- 월간/기간별 API 호출량 (요청 수, 에러 비율)
- 스토리지 사용량 (GB)
- 시뮬레이션 실행 수 (`scenarios` 카운트, 타입별 분포)
- `simulation_results` 수, 실패율
- 대시보드 집계·조회 호출량
- (선택) 플랜 정보/라이선스 정보 요약
  - 현재 플랜 타입 (Free / Pro / Enterprise 등)
  - 등록 매장 수, 활성 사용자 수

**정렬/검색**

- 기간 선택 (커스텀 날짜 범위)
- 기능 타입별 탭 (API / Simulation / Dashboard 등)

**관리 액션**

- 사용량 리포트 다운로드 (해당 테넌트 전용 CSV/PDF)
- 특정 기능(예: 과도한 Simulation 사용)에 메모/태그 추가
- 이상 패턴 감지 시 AI 및 시뮬레이션 모니터링, 시스템 및 오류 관리로 바로가기

---

### 3.3 데이터 관리

#### 3.3.1 목적

- 데이터 임포트/동기화/집계 파이프라인의 상태를 한눈에 파악한다.
- 각 테넌트의 데이터 품질(Data Health)을 평가하고 문제 지점을 빠르게 찾을 수 있도록 한다.

#### 3.3.2 화면 구조

1. **Data Pipeline Overview (전역 뷰)**
   - 라우트: `/hq/data-pipeline`
   - 전체 Import/Sync/집계 파이프라인 상태 요약.

2. **Tenant Data Health Detail**
   - 라우트: `/hq/data-pipeline/:tenantId`
   - 단일 테넌트에 대한 Data Health 상세/에러 로그/재시도 액션 제공.

#### 3.3.3 Data Pipeline Overview 뷰

**노출 데이터**

- 최신 `user_data_imports` 성공/실패 로그 요약
- 외부 데이터 동기화 (`data_sync_schedules`, `data_sync_logs`) 상태
- WiFi/IoT 데이터 수신 상태
  - 디바이스 마지막 수신 시간
  - offline 여부

- 전역 Data Health 지표
  - 최근 N일 평균 Import 성공률
  - 주요 테이블별 평균 데이터 볼륨 변화(전 테넌트 합산)
  - 실패 Job 상위 유형

**정렬**

- 최신/오래된 Job 순 정렬
- Job 타입별 그룹화(Import/Sync/Aggregation)

**관리 액션**

- 실패 Job 상세 화면 이동 (Tenant Data Health Detail)
- 전체 Import/Sync 재시도 트리거(제한적, 운영자에게 경고 표시)
- 에러 유형별 메모/태그 추가

#### 3.3.4 Tenant Data Health Detail 뷰

**노출 데이터**

- 최근 N일 임포트/동기화 성공율
- 주요 테이블별 데이터 볼륨 변화 추이
  - 예: visits, sales, wifi_tracking 등
- 에러 유형 분포
  - 형식 오류, 누락 컬럼, 타임아웃, 인증 실패 등
- Data Health Score (내부 기준에 따른 스코어링)
  - 예: A/B/C 등급 또는 0–100 점수

- 실패 Job 리스트
  - Job ID, Job 타입
  - 실행 시각, 대상 스토어/기간
  - 에러 메시지 요약
  - (선택) 예시 에러 레코드 샘플

**정렬**

- 실행 시각 기준 정렬
- 에러 유형 기준 그룹화

**관리 액션**

- 실패 Job 재시도
- 대시보드 KPI 재집계 요청 (해당 스토어/기간 대상)
- Data Health 리포트 출력 (CSV/PDF)
- 온톨로지 스키마 관리로 이동하여 스키마 관련 이슈 확인

---

### 3.4 온톨로지 스키마 관리

> 중요
> - 고객(테넌트)용 애플리케이션에서는 Graph 구조가 직접 노출되지 않는다.
> - 고객은 임포트 → 분석 결과/시뮬레이션 결과만 보며,
>   Ontology 스키마 및 Graph Entities/Relations는 관리자 전용 뷰에서만 조회·관리된다.

#### 3.4.1 목적

- Ontology 스키마(Entity/Relation 타입) 정의·버전·변경 이력을 관리한다.
- Graph 인스턴스(`graph_entities`, `graph_relations`)의 상태·규모·품질을 모니터링한다.
- 스키마 변경이 각 테넌트 데이터에 미치는 영향을 관리한다.

#### 3.4.2 화면 구조

1. **Ontology Schema**
   - 라우트: `/hq/ontology`
   - 엔티티/관계 타입 정의, 속성, 버전 이력 관리.

2. **Global Graph Monitoring**
   - 라우트: `/hq/graph`
   - 전역 Graph 개요 및 품질 지표 확인.

3. **Tenant Graph Detail**
   - 라우트: `/hq/graph/:tenantId`
   - 테넌트별 Graph 인스턴스 상태, 품질 문제 점검.

#### 3.4.3 Ontology Schema 뷰

**노출 데이터**

- `ontology_entity_types`
  - 엔티티 타입명, 레이블
  - 속성 목록 (필수/선택, 타입, 단위 등)
  - 3D 연관 메타데이터 (3D Asset 타입, Placement 규칙 등)

- `ontology_relation_types`
  - 관계 타입명, 레이블
  - Source/Target 엔티티 타입
  - 방향성 정보 (단방향/양방향)
  - 제약 조건 (카디널리티 등)

- 스키마 버전/변경 이력
  - 버전 ID, 적용 일시
  - 변경 요약, 영향 범위

**관리 액션**

- 엔티티/관계 타입 추가/수정/비활성화
- 스키마 변경 초안 작성 및 저장(임시 상태)
- 변경 승인 및 배포(버전 증가)
- 특정 버전 기준 스키마 diff 보기

#### 3.4.4 Global Graph Monitoring 뷰

**노출 데이터**

- 전역 Graph 개요
  - 전체 노드 수, 관계 수
  - 엔티티 타입별 분포 (노드 수)
  - 관계 타입별 분포 (엣지 수)

- 품질 메트릭
  - 관계 없는 고립 노드 비율
  - 필수 속성 누락 비율
  - 스키마 위반(예: 정의되지 않은 타입 간 관계) 카운트

- 테넌트별 요약
  - 각 테넌트 노드/엣지 수
  - 각 테넌트의 Graph Health Score

**관리 액션**

- 특정 테넌트 Graph 상세로 이동
- 품질 메트릭 기준 상위 이슈 테넌트 하이라이트
- Graph 정합성 검사 Job 실행 (전역/부분)

#### 3.4.5 Tenant Graph Detail 뷰

**노출 데이터**

- 테넌트별 Graph 개요
  - 노드 수, 관계 수
  - 엔티티 타입별 분포(예: Product, Zone, Fixture 등)
- 특정 엔티티 타입 중심 부분 그래프 조회
  - 예: 특정 매장 Zone 주변 관계 구조
- 품질 메트릭
  - 고립 노드 리스트
  - 필수 속성 누락 엔티티 리스트
  - 스키마 위반 관계 리스트

**관리 액션**

- 정합성 검사 실행(해당 테넌트만)
- 비정상 인스턴스 목록 Export
- 자동 정리 Job 트리거(또는 후보만 마크)
- 3D 디지털 트윈 관리로 이동하여 Scene/Graph 매핑 상태 확인

---

### 3.5 3D 디지털 트윈 관리

> 이 섹션은 **두 개의 파트**로 구성된다.
> 1) 사용자가 제출한 디지털 트윈 3D 씬 제작 요청을 관리하는 파트  
> 2) 관리자가 3D 에셋과 데이터를 업로드하고 씬을 저장·배포하는 파트

#### 3.5.1 목적

- 고객(테넌트)로부터 들어온 3D 디지털 트윈 제작 요청을 체계적으로 관리한다.
- 관리자가 자체적으로 3D 씬을 구성하고, 적절한 요청/테넌트에 배포할 수 있는 워크스페이스를 제공한다.
- 테넌트 대시보드의 디지털 트윈 3D 씬 관리와 최대한 동일한 UX로, 관리자가 대신 작업하는 형태를 지원한다.

#### 3.5.2 화면 구조

1. **제작 요청 관리**
   - 라우트: `/hq/3d-requests`
   - 여러 사용자들로부터 들어온 3D 디지털 트윈 제작 요청 목록을 관리하고,
     각 요청의 상세/첨부 파일/응답 내역을 확인·처리하는 화면.

2. **3D 프로덕션 관리**
   - 라우트: `/hq/3d-studio`
   - 관리자용 3D 편집 워크스페이스.
   - 테넌트 대시보드의 디지털 트윈 3D 씬 관리 탭과 거의 동일한 구성:
     - 모델 레이어 관리
     - 씬 저장
     - 3D 프리뷰
   - 상단에 “배포하기” 버튼을 두어, 현재 3D 프리뷰 씬을 어떤 Tenant/어떤 요청에 할당할지 선택하고 배포.

---

#### 3.5.3 제작 요청 관리

##### 3.5.3.1 요청 목록 뷰

**목적**

- 여러 사용자로부터 접수된 제작 요청을 한눈에 파악하고,
  상태 조정·상세 확인·응답을 빠르게 수행한다.

**라우트**

- `/hq/3d-requests`

**노출 데이터 (컬럼)**

- **Tenant**
  - 요청을 생성한 테넌트 식별 정보 (조직명/브랜드명)
- **프로젝트 ID**
  - 요청별로 자동 부여되는 ID (내부 PK와 별도 노출용 ID 가능)
- **프로젝트 이름**
  - 요청 시 사용자가 기입한 프로젝트 이름
- **상태(Status)**
  - `Requested` / `InProduction` / `InReview`
  - 관리자가 Combobox로 직접 선택하여 변경
  - 상태 변경 시, 관리자 콘솔 뿐만 아니라
    **사용자 대시보드의 요청 상세 화면에서도 동일한 상태가 노출**되어야 함
  - 요청이 처음 생성될 때의 초기 상태는 항상 `Requested`
- **요청 생성일**
  - 요청이 생성된 일시
- **배포 여부**
  - 이 요청에 대응하는 3D 씬이 실제 테넌트 디지털 트윈에 할당되었는지 여부 (예/아니오)

**정렬 기능**

- 상태 기준 뷰
  - “Requested만 보기”
  - “InProduction만 보기”
  - (UI적으로는 토글/탭 형태의 상태별 뷰 제공, 일반적인 다중 필터 UI는 불필요)
- 요청 생성일 기준 정렬
  - 가장 최근 날짜 순
  - 가장 오래된 날짜 순

**행별 공통 액션**

- **상세 보기**
  - 클릭 시 별도 라우트로 이동하지 않고,
    같은 화면 내에서 상세 패널/오버레이/슬라이드 패널 형태로 전환
  - 뒤로가기(← 화살표) 버튼을 제공하여 목록 화면으로 복귀
- **응답**
  - 클릭 시 “응답 작성/히스토리” 패널로 전환
  - 마찬가지로 뒤로가기(← 화살표) 버튼으로 목록으로 돌아가기

> 구현 관점:
> - `/hq/3d-requests` 단일 라우트 내에서
>   `mode = list | detail | reply` 형태로 UI 상태를 전환하는 구조를 권장.
> - URL 쿼리(`?projectId=...&view=detail`) 정도는 사용 가능하나,
>   별도의 다른 path로 라우팅 전환까지는 필요 없음.

---

##### 3.5.3.2 요청 상세 보기 뷰 (상세 보기)

**진입 경로**

- 제작 요청 관리 목록에서 특정 행의 “상세 보기” 버튼 클릭 시

**노출 데이터**

- 상단 헤더
  - Tenant
  - 프로젝트 ID
  - 프로젝트 이름
  - 상태 (Combobox, 여기서도 변경 가능)
  - 요청 생성일
  - 배포 여부

- **요청 정보 섹션**
  - 사용자가 요청 시 기입한 프로젝트 설명
  - 매장 유형, 면적, 층수, 주요 존 설명 등 (요청 폼에 포함된 필드들)
  - 특이사항(브랜드 톤, 강조 구역 등)

- **첨부 파일 섹션**
  - 사용자가 업로드한 파일 목록
    - 파일명, 타입(PDF/이미지 등), 업로드 일시
  - 관리자용 다운로드 버튼
    - 외부 3D 툴(Blender, 3ds Max 등)에서 참조하기 위한 원본 도면/사진 파일 확보용

**UI 동작**

- 화면 상단 좌측에 “← 뒤로가기” 버튼
  - 클릭 시 요청 목록 뷰로 복귀
- 동일 패널/화면 내에서 “응답 보기/쓰기로 전환” 버튼을 둬도 됨
  - 클릭 시 아래 3.5.3.3 응답 뷰로 전환

---

##### 3.5.3.3 요청 응답 뷰 (응답)

**진입 경로**

- 목록에서 “응답” 버튼 클릭
- 또는 상세 보기 뷰에서 “응답 쓰기” 버튼 클릭

**노출 데이터**

- 상단 헤더
  - Tenant, 프로젝트 ID, 프로젝트 이름 (간략 표시)
  - 현재 상태 (읽기/편집 가능 Combobox – 선택 사항)

- **응답 작성 섹션**
  - 텍스트 입력 영역
  - “보내기” 버튼
  - 입력한 메시지는 관리자→사용자 방향 커뮤니케이션으로 저장되며,
    사용자 대시보드의 “요청 상세/메시지” 영역에서도 동일하게 노출

- **응답 히스토리 섹션**
  - 과거에 관리자/사용자 간 주고받은 메시지 리스트
  - 발신자(관리자/사용자), 시간, 내용
  - 최신 순 정렬

**UI 동작**

- 상단 좌측 “← 뒤로가기” 버튼
  - 클릭 시 요청 목록 뷰로 복귀
- 필요 시 “요청 상세 보기로 전환” 버튼을 두어
  - 동일 라우트 내에서 상세/응답 뷰 간 스위칭 가능

---

#### 3.5.4 3D 프로덕션 관리

> 사용자 대시보드의 디지털 트윈 3D 씬 관리 UI를 거의 그대로 차용한다.  
> 구성 요소는 다음과 같다.
> - 모델 레이어 관리
> - 씬 저장
> - 3D 프리뷰
> 상단에는 “배포하기” 버튼이 존재한다.

##### 3.5.4.1 모델 레이어 관리

**목적**

- 관리자가 로컬에서 준비한 3D 모델(glb/gltf)과 데이터(CSV/JSON)를 업로드하고,
  논리적인 레이어(매장공간/가구/상품) 단위로 관리한다.

**노출 요소**

- “3D 모델 및 데이터 업로드” 버튼
  - 클릭 시 OS 파일 선택 창(윈도우 탐색기 등) 오픈
  - 허용 파일 타입:
    - 3D 모델: `.glb`, `.gltf`
    - 데이터: `.csv`, `.json`

- 업로드된 파일 리스트
  - **3개 카테고리로 분류되어 출력**
    - `매장공간`
    - `가구`
    - `상품`
  - 각 카테고리별로 업로드된 파일 목록
    - 파일 이름
    - 파일 타입(glb/gltf/csv/json)
    - 업로드 일시
    - (선택) 적용 여부 토글, 삭제 버튼 등

**카테고리 분류 방식 예시**

- 업로드 시 관리자에게 카테고리 선택 Combobox 제공
  - 매장공간 / 가구 / 상품 중 하나 선택 필수
- 또는 파일명 규칙/메타데이터를 기반으로 자동 제안 + 수정 가능

**데이터 측면**

- 업로드된 파일들은 3D 전용 스토리지 버킷(예: `3d-assets`)에 저장
- 메타데이터 테이블에서
  - 파일 경로
  - 카테고리(매장공간/가구/상품)
  - 연관 테넌트/요청(선택)
  등을 관리

---

##### 3.5.4.2 씬 저장

**목적**

- 현재 모델 레이어 구성을 “씬(Scene)” 단위로 저장·업데이트한다.

**노출 요소**

- 씬 이름 입력 필드
  - 새 씬 생성 시: 이름 입력 후 저장
  - 기존 씬 수정 시: 이름 고정 or 변경 가능 (정책에 따라)
- “씬 업데이트” 버튼
  - 클릭 시 현재 레이어 구성 상태를 SceneRecipe 형태로 저장/업데이트
  - 저장 후, 최신 SceneRecipe를 기반으로 3D 프리뷰를 갱신

**데이터 측면**

- SceneRecipe는 `store_scenes.recipe_data` 유사 구조로 저장
  - 단, 이 단계에서는 특정 테넌트/스토어에 아직 할당되지 않은 “템플릿 씬” 개념이어도 됨
- 나중에 “배포하기” 단계에서
  - 특정 Tenant/요청/스토어에 이 SceneRecipe를 연결

---

##### 3.5.4.3 3D 프리뷰 & 배포하기

**3D 프리뷰**

- Three.js 기반 렌더링 영역
- 현재 저장된 SceneRecipe를 렌더링
- 사용자 대시보드의 디지털 트윈 3D 프리뷰와 동일한 카메라 조작/레이어 On/Off 제공
- 레이어별 가시성 토글 (매장공간/가구/상품 및 세부 레이어)

**배포하기 버튼 (상단 공통)**

- 버튼 라벨: “배포하기”
- 클릭 시 작은 팝업/모달 창 오픈

**배포 팝업에서의 안내 및 입력**

- 안내 문구
  - “현재 3D 프리뷰에서 보이는 씬을 어떤 Tenant/요청에 할당하시겠습니까?”
- 선택 항목
  1. **Tenant 선택**
     - 드롭다운 또는 검색 가능한 선택 박스
     - 테넌트 목록 표시
  2. **요청 선택**
     - 선택된 Tenant의 3D 제작 요청 목록을 표시
     - (3.5.3의 요청 목록과 동일한 데이터 구조)
     - 프로젝트 ID + 프로젝트 이름을 함께 표시
- “할당하기” 버튼
  - 클릭 시 다음 처리가 수행된다.
    - 현재 SceneRecipe를 선택된 요청/테넌트에 연결
    - 해당 요청의 “배포 여부”를 `예`로 업데이트
    - 필요 시 요청 상태를 자동으로 `InReview` 등으로 변경하는 정책 적용 가능
    - 테넌트 대시보드의 디지털 트윈 3D 화면에
      이 SceneRecipe가 연결되도록 `store_scenes` 등 관련 테이블 업데이트

**배포 후 결과**

- 관리자 콘솔
  - 배포 성공 토스트/알림 표시
  - 관련 요청(3.5.3)의 “배포 여부” 컬럼 업데이트
- 사용자 대시보드
  - 디지털 트윈 3D 메뉴 진입 시,
    방금 배포된 SceneRecipe를 기반으로 3D 프리뷰가 렌더링

---

#### 3.5.5 데이터 연동 & 저장소 구조 (요약)

**스토리지**

- 3D 전용 버킷 예시: `3d-assets`
  - 공용 Asset: `3d-assets/global/...`
  - 테넌트 전용 Asset: `3d-assets/tenant/{tenant_id}/...`
  - (필요 시) 매장 전용 Asset: `3d-assets/tenant/{tenant_id}/store/{store_id}/...`
- 테넌트 대시보드 데이터 임포트(`user_data_imports`)용 스토리지(`store-data` 등)와 논리적으로 분리

**Scene & Graph 연동**

- SceneRecipe 저장
  - 관리용 템플릿 씬 또는 특정 테넌트/스토어에 연결된 씬 모두
  - `store_scenes` 또는 별도 `admin_scenes` 테이블로 관리 가능
- Graph 연동
  - SceneRecipe 내 인스턴스 정보(tenant/store, 비즈니스 키, 위치/스케일 등)를 이용해
    `graph_entities` 생성/업데이트
  - 테넌트가 user data import로 업로드한 데이터 기반 Graph 노드와,
    관리자가 3D 씬을 통해 만든 Graph 노드를
    동일 Ontology 스키마 내에서 통합 관리

**요청과의 연동**

- 3D 제작 요청 메타데이터 테이블 예시: `admin_scene_projects`
  - Tenant, 프로젝트 ID, 상태, 요청 생성일, 배포 여부 등 저장
- 응답 메시지 히스토리 테이블 예시: `admin_scene_project_messages`
- 배포 시:
  - `admin_scene_projects`의 배포 여부/상태 업데이트
  - 해당 요청과 SceneRecipe/`store_scenes`를 연결하는 키 저장

---

### 3.6 AI 및 시뮬레이션 모니터링

#### 3.6.1 목적

- 각 테넌트의 시뮬레이션/AI 기능 사용 현황과 실패율을 모니터링한다.
- 모델/파이프라인 수준에서의 문제를 조기에 감지한다.

#### 3.6.2 화면 구조

1. **AI & Simulation Overview**
   - 라우트: `/hq/ai-sim`
   - 전역 시뮬레이션/AI 실행 현황 요약.

2. **Tenant AI & Simulation Detail**
   - 라우트: `/hq/ai-sim/:tenantId`
   - 테넌트별 시뮬레이션/AI 실행 이력, 실패율, 응답 시간 상세.

#### 3.6.3 AI & Simulation Overview 뷰

**노출 데이터**

- 전체 시뮬레이션 실행 수 (기간별)
- 실패율 (에러 코드/타입 별)
- 평균 응답 시간
- 시뮬레이션 타입별 분포
  - layout / pricing / demand / recommendation 등
- AI 관련 호출 요약
  - `ai_recommendations`, `ai_scene_analysis` 등 카운트

**관리 액션**

- 실패율 상위 테넌트 목록 확인
- 특정 테넌트/타입 선택 시 Tenant Detail로 이동
- 에러 코드/타입에 메모/태그 추가

#### 3.6.4 Tenant AI & Simulation Detail 뷰

**노출 데이터**

- 해당 테넌트의 시뮬레이션 실행 이력
  - `scenarios` (타입별: layout/pricing/demand/recommendation)
  - 실행 시각, 입력 파라미터 요약, 상태(성공/실패)
- `simulation_results` 요약
  - 결과 생성 여부, 주요 KPI 변화량 등
- AI 관련 호출
  - `ai_recommendations`, `ai_scene_analysis` 횟수/실패율
- 대표 지표
  - 시뮬레이션 타입별 실패율
  - 글로벌 평균 대비 실패율/응답시간 이상 여부

**관리 액션**

- 실패한 시나리오 상세 로그 열람
- 특정 시나리오 재실행 요청
- 특정 테넌트/기능에 대한 임시 제한 설정 (예: Pricing Simulation 임시 중단)
- 모델 버전/설정 변경 이력 조회 (외부 모델 관리 시스템과 연동 시)

---

### 3.7 시스템 및 오류 관리

#### 3.7.1 목적

- Edge Functions, API, Frontend 오류 등 시스템 전반의 에러를 중앙에서 모니터링한다.
- 운영자가 문제의 위치(테넌트별/기능별/시간대별)를 빠르게 추적할 수 있도록 한다.

#### 3.7.2 화면 구조

1. **Error & Performance Overview**
   - 라우트: `/hq/system/errors`
   - 에러/HTTP 상태 코드/응답 시간 등 요약.

2. **Edge Functions Status**
   - 라우트: `/hq/system/functions`
   - Edge Functions별 호출 수/실패율/응답 시간 모니터링.

#### 3.7.3 Error & Performance Overview 뷰

**노출 데이터**

- HTTP 에러 로그 요약
  - 4xx/5xx 비율, 엔드포인트별 분포
- 성능 트렌드
  - 평균 응답 시간, p95, 특정 시간대 부하 집중 여부
- 에러 Hotspot
  - 특정 테넌트/엔드포인트/기능 조합에서 에러 집중 여부

**관리 액션**

- 에러 로그 필터링 (테넌트/함수/HTTP 코드/시간대)
- 에러 세부 정보 확인 (Request/Response 요약, Trace ID, 관련 테넌트)
- 반복되는 에러 유형에 태그/메모 추가

#### 3.7.4 Edge Functions Status 뷰

**노출 데이터**

- Edge Function별 호출 수
- 실패율
- 평균/최대 응답 시간
- 최근 배포/버전 변경 이력(선택)

**관리 액션**

- 특정 함수 에러 로그 상세로 이동
- 특정 함수 호출 제한/비활성화(정책에 따라)
- 배포 이력/릴리즈 노트 링크

---

### 3.8 관리자 도구

#### 3.8.1 목적

- 운영자가 반복적으로 수행하는 관리 작업을 쉽게 실행할 수 있도록 도구화한다.
- 기능 플래그/실험 설정을 중앙에서 관리한다.

#### 3.8.2 화면 구조

1. **Admin Tools**
   - 라우트: `/hq/tools`
   - 재집계/리빌드/테스트용 유틸리티 모음.

2. **Admin Settings**
   - 라우트: `/hq/settings`
   - Admin 계정/권한, Feature Flag, Impersonation 정책 등의 설정.

#### 3.8.3 Admin Tools 뷰

**노출 기능**

- Feature Flag 상태 요약(읽기 전용 링크)
- 강제 재집계/리빌드
  - 대시보드 KPI 재집계 트리거
  - 특정 기간/스토어에 대한 Graph 재생성(재임포트 기반) 요청
- 테스트 유틸리티
  - 외부 API 연결 상태 점검
  - 특정 테넌트/스토어에 대한 Sanity Check 실행

**관리 액션**

- 재집계 Job 실행
- Graph 재생성 Job 실행
- 테스트 결과 로그 확인

#### 3.8.4 Admin Settings 뷰

**노출 데이터**

- Admin 계정 목록
  - 계정, 역할(Role: `platform_admin`, `ops_admin`, `read_only_admin` 등)
  - 상태(Active/Disabled)
- Feature Flags 설정
  - 플래그 키, 설명
  - 기본값, 테넌트/플랜별 Override 설정
- Impersonation 정책
  - 허용 여부
  - 읽기 전용/쓰기 허용 범위

**관리 액션**

- Admin 계정 생성/수정/비활성화
- 기능 플래그 On/Off (전역/테넌트별)
- Impersonation 권한 범위 설정

---

## 4. 운영 시나리오 & 대응 플로우

### 4.1 시나리오 1 – 데이터 임포트 실패 문의

1. **사용자 증상**
   - “CSV를 업로드했는데 계속 실패합니다.”

2. **관리자 플로우**
   - 사용자 관리 → 해당 테넌트 검색 → 상세 화면 진입
   - “최근 데이터 임포트” 위젯에서 실패 건 클릭
   - 데이터 관리 → 해당 Job 상세 보기
     - 오류 유형(필수 컬럼 누락/형식 오류/타임아웃 등) 확인
   - 필요 시 “재시도” 실행
   - 문제가 스키마/Ontology와 관련된 경우
     → 온톨로지 스키마 관리에서 해당 데이터 타입 스키마 점검
   - 처리 결과를 CS 시스템 또는 내부 메모에 기록

---

### 4.2 시나리오 2 – 대시보드 숫자 이상 문의

1. **사용자 증상**
   - “전날보다 유입이 갑자기 0으로 나오는데, 실제로는 매장이 열려 있었습니다.”

2. **관리자 플로우**
   - 사용자 관리 → 해당 테넌트 선택
   - 데이터 관리 → 해당 날짜 데이터 Health 확인
     - WiFi/Visits 데이터 누락/지연 여부 확인
   - 외부 API(공휴일/이벤트) 동기화 상태 확인
   - 경우에 따라 KPI 집계 함수 에러 여부를 시스템 및 오류 관리에서 추가 확인
   - 문제가 파악되면 “해당 기간 대시보드 재집계” 실행
   - 재집계 이후, 수치 정상화 여부를 직접 확인 후 사용자에게 안내

---

### 4.3 시나리오 3 – 시뮬레이션 실패 문의

1. **사용자 증상**
   - “어제부터 레이아웃 시뮬레이션이 계속 에러가 납니다.”

2. **관리자 플로우**
   - AI 및 시뮬레이션 모니터링 → 해당 테넌트 필터
   - 실패 시나리오 목록에서 최근 레코드 확인
   - 관련 Edge Function 로그/오류 메시지 확인
   - 파라미터/데이터/모델 응답 중 어느 영역 문제인지 분류
   - 단기 조치:
     - 특정 시나리오 재실행
     - 동일 유형 시뮬레이션 임시 제한(문제 확산 방지)
   - 장기 조치:
     - 데이터/모델/코드 수정 후 릴리즈
     - 재테스트 및 모니터링

---

### 4.4 시나리오 4 – 3D 디지털 트윈 제작 요청 및 배포

1. **사용자 상황**
   - “우리 매장 3D 모델이 없는데, 디지털 트윈을 사용하고 싶습니다.”
   - 공식 웹사이트의 3D 디지털 트윈 문의/의뢰 폼(또는 유사한 외부 채널)을 통해
     요청을 제출하면, 백엔드에서 해당 테넌트 기준의 3D 프로젝트가
     상태 `Requested` 로 생성된다.
   - 이때 업로드된 도면/사진/스케치 파일은 3D 제작 참고 자료로 함께 저장된다.

2. **관리자 플로우**
   - 제작 요청 관리 → 요청 목록에서 상태 `Requested` 인 항목 확인
   - 해당 요청의 “상세 보기” 진입
     - 매장 도면/사진/필수 정보 검수
     - 누락 정보가 있을 경우 테넌트 측에 보완 요청
     - 필요한 참고 자료(도면/사진)를 로컬로 다운로드
   - 필요 시 “응답” 화면에서 추가 설명/요청 사항 전달
   - 내부/외주 3D 팀이 외부 3D 툴에서 모델링/텍스처링/베이크 작업 수행
     - 완성된 3D 모델을 `glb` 형식으로 추출
   - 관리자 콘솔의 3D 프로덕션 관리 화면에서
      - `glb`/`gltf` 및 관련 CSV/JSON 데이터를 업로드
      - 모델 레이어 관리에서 매장공간/가구/상품으로 분류
      - SceneRecipe 구성 후 “씬 업데이트”로 저장
   - 3D 프리뷰에서 씬이 정상적으로 보이는지 확인 후
     - 상단 “배포하기” 버튼 클릭
     - 팝업에서 Tenant 및 해당 요청을 선택하고 “할당하기” 실행
   - 요청의 “배포 여부”가 `예`로 업데이트되고,
     테넌트 대시보드의 디지털 트윈 3D 화면에 씬이 노출

---

### 4.5 시나리오 5 – Ontology 스키마 변경 필요

1. **내부 요구사항**
   - 새로운 엔티티 타입/속성을 추가해야 하거나
     기존 타입 구조가 비즈니스 요구에 맞지 않는 상황.

2. **관리자 플로우**
   - 온톨로지 스키마 관리 → 스키마 설계 변경
   - 영향 받는 테넌트/데이터 유형 파악
   - 제한된 범위에서 사전 테스트용 테넌트/스토어 선정
   - 변경 적용 후 데이터 관리, Graph Monitoring 을 통해 품질 점검
   - 문제 없을 시 전체 테넌트에 순차 적용
   - 스키마 버전 및 변경 이력 기록

---

## 5. Supabase 공통 백엔드 & 권한 아키텍처

### 5.1 전체 컨셉 한 줄 요약

> **Supabase 프로젝트 1개 = NEURALTWIN 공통 백엔드**

이 하나의 Supabase 프로젝트를

- **Website (Public)**
- **Customer Dashboard (고객용 앱)**
- **HQ Admin (관리자 콘솔)**

이 세 개 Lovable 프론트엔드에서 **같은 Auth / DB / Edge Functions** 로 접근하는 구조다.

- Website:  
  유저·조직·구독을 **만드는 입구**
- Customer Dashboard:  
  조직이 실제로 **매장/데이터/시뮬레이션을 돌리는 운영 도구**
- HQ Admin:  
  NEURALTWIN 팀이 **모든 테넌트·온톨로지·파이프라인·모델·비즈니스를 컨트롤하는 타워**

---

### 5.2 Supabase 프로젝트 1개 만들기

1. Supabase 콘솔에서 **새 프로젝트 1개** 생성  
2. 이 프로젝트를 Website / Customer Dashboard / HQ Admin **전 채널의 공통 백엔드**로 사용

3개 Lovable 프로젝트에 **동일한 환경변수**를 셋업한다.

```env
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_ANON_KEY=public-anon-key
SUPABASE_SERVICE_KEY=service-role-key  # 서버/Edge Functions 전용
```

- Website / Customer / HQ 모두 **같은 URL/ANON_KEY**를 쓰는 것이 핵심
- Edge Functions나 백오피스 스크립트에서는 `SERVICE_KEY` 사용

각 프론트엔드에서는 대략 이런 형태로 Supabase Client를 만든다.

```ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```

---

### 5.3 Auth & 멀티테넌시 설계

Supabase는 기본적으로 `auth.users` 테이블을 제공하므로,  
여기에 우리가 필요한 **조직(Brand) / 역할 / 라이선스** 레이어만 얹는다.

#### 5.3.1 핵심 테이블 구조 (개념)

```sql
-- 조직 (브랜드/기업 단위)
create table organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz default now()
);

-- 조직 구성원 (유저↔조직 관계 + 역할)
create table organization_members (
  org_id uuid references organizations(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  role text check (role in ('ORG_OWNER','HQ_ANALYST','STORE_MANAGER','VIEW_ONLY','NEURALTWIN_ADMIN')),
  created_at timestamptz default now(),
  primary key (org_id, user_id)
);

-- 구독/플랜 (Stripe/PG 연동 결과)
create table subscriptions (
  id uuid primary key default gen_random_uuid(),
  org_id uuid references organizations(id) on delete cascade,
  provider text not null,          -- 'stripe' or 'toss' etc.
  provider_sub_id text not null,   -- stripe subscription id
  plan text not null,              -- 'starter','pro','enterprise'
  status text not null,            -- 'active','past_due','canceled',...
  current_period_end timestamptz,
  created_at timestamptz default now()
);

-- 라이선스/쿼터 (매장 수/HQ seat 수 등)
create table licenses (
  org_id uuid references organizations(id) primary key,
  max_stores int not null default 1,
  max_hq_seats int not null default 1,
  max_events_per_month int,
  created_at timestamptz default now()
);
```

운영 데이터(Stores, Products, Customers, Visits, Purchases, Inventory …)에도  
**반드시 `org_id` 컬럼을 포함**하여 멀티테넌시를 유지한다.

```sql
create table stores (
  id uuid primary key default gen_random_uuid(),
  org_id uuid references organizations(id) on delete cascade,
  store_code text not null,
  name text not null
  -- ...
);
```

#### 5.3.2 RLS(Row Level Security)로 테넌트 격리

Supabase의 강점은 **RLS** 이므로,  
멀티테넌트 테이블에는 전부 RLS를 켜고 정책을 걸어준다.

```sql
alter table organizations enable row level security;
alter table organization_members enable row level security;
alter table stores enable row level security;
-- etc...
```

정책 예시 – “내가 멤버인 조직만 조회 가능”:

```sql
create policy "members can select their org"
on organizations
for select using (
  exists (
    select 1 from organization_members om
    where om.org_id = organizations.id
      and om.user_id = auth.uid()
  )
);
```

이렇게 하면 Website / Customer / HQ 가 **모두 같은 DB**를 쓰더라도,  
JWT 안의 `user_id`와 `org_id`에 따라 각 채널에서 볼 수 있는 데이터가 자동으로 제한된다.

- **Customer Dashboard**  
  - 보통 `ORG_OWNER / HQ_ANALYST / STORE_MANAGER / VIEW_ONLY` 역할
- **HQ Admin**  
  - `NEURALTWIN_ADMIN` 역할일 때  
    모든 org를 조회할 수 있게 하는 **별도 RLS Policy**를 추가로 둔다  
  - 예: `role = 'NEURALTWIN_ADMIN'` 이면 `org_id` 필터 없이 select 허용

---

### 5.4 공통 API / Edge Functions 계층

Supabase는 기본 REST + RPC를 제공하지만,  
비즈니스 로직/AI/시뮬레이션은 **Edge Functions** 레이어로 감싼다.

- 간단 CRUD → Supabase Table/REST 직접 사용
- 복잡한 로직 / 통합 처리 / AI 호출 → Edge Functions (Deno) 사용

#### 5.4.1 가입/조직 생성용 Edge Function (Website → 공통 Org 생성)

Website에서 회원가입을 마친 뒤,  
조직과 라이선스를 자동으로 만드는 Edge Function을 호출한다.

프론트(Website) 예시:

```ts
const { data: { user }, error } = await supabase.auth.signUp({ email, password });

if (user) {
  await fetch('/functions/v1/create-org-and-license', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${user.access_token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      org_name: companyName,
      plan: 'starter'
    })
  });
}
```

Edge Function 개념 예시:

```ts
// supabase/functions/create-org-and-license/index.ts
import { serve } from "https://deno.land/std/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js";

serve(async (req) => {
  const supabase = createClient(DENO_SUPABASE_URL, DENO_SUPABASE_SERVICE_KEY);
  const authHeader = req.headers.get("Authorization") || "";
  const jwt = authHeader.replace("Bearer ", "");

  const { data: { user }, error: authError } = await supabase.auth.getUser(jwt);
  if (authError || !user) return new Response("Unauthorized", { status: 401 });

  const { org_name, plan } = await req.json();

  // org 생성
  const { data: org, error: orgError } = await supabase
    .from("organizations")
    .insert({ name: org_name })
    .select()
    .single();

  // 멤버 연결
  await supabase.from("organization_members").insert({
    org_id: org.id,
    user_id: user.id,
    role: "ORG_OWNER"
  });

  // 기본 라이선스
  await supabase.from("licenses").insert({
    org_id: org.id,
    max_stores: plan === "pro" ? 5 : 1,
    max_hq_seats: plan === "pro" ? 5 : 1
  });

  return new Response(JSON.stringify(org), { status: 200 });
});
```

→ Website는 “유저 생성 + Edge Function 호출”까지만 하면 되고,  
Customer/HQ는 DB에서 동일 `org_id` 정보를 그대로 읽어서 사용한다.

#### 5.4.2 시뮬레이션/AI용 Edge Function (NEURALTWIN Core)

이미 사용 중인 `advanced-ai-inference`, `integrated-data-pipeline` 등이 여기에 해당한다.

공통 API 형태 예시:

```http
POST /functions/v1/advanced-ai-inference

{
  "scenario_type": "layout" | "demand" | "inventory" | "pricing" | "promotion",
  "org_id": "…",
  "store_id": "A001",
  "params": { ... } // 모듈별 파라미터
}
```

Edge Function 내부에서:

1. `org_id`, `store_id` 기반으로 Supabase DB에서 NEURALMIND 데이터 조회  
2. AI/모델 로직 실행  
3. ΔKPI, 추천안, 로그를 반환/저장

- Customer Dashboard:
  - 이 함수를 직접 호출해 시뮬레이션을 실행
- HQ Admin:
  - 이 함수 자체를 호출하기보다는  
    “실행 이력/실패율/응답시간”을 모니터링하는 쪽에 집중

---

### 5.5 채널별 역할 분리 & HQ 권한 모델

#### 5.5.1 채널별 책임 정리

- **Website 프로젝트**
  - `supabase.auth`로 회원가입/로그인 처리
  - 회원가입 후 `create-org-and-license` Edge Function 호출
  - 결제 Webhook(Stripe/PG)에서
    - `subscriptions`, `licenses` 업데이트 책임

- **Customer Dashboard 프로젝트**
  - 로그인 시 Supabase JWT 확인
  - `organization_members`에서 사용자의 org/role 로딩
  - 모든 API/쿼리에 `org_id` 필터 적용 (또는 BFF에서 자동 필터)
  - Analysis / Simulation / Digital Twin 화면에서
    - Supabase 테이블 + Edge Functions 호출

- **HQ Admin 프로젝트**
  - 동일 Auth를 쓰지만, `role = 'NEURALTWIN_ADMIN'` 또는 `is_internal = true` 만 접근 허용
  - `organizations`, `subscriptions`, `licenses`, `ontology_*`, `pipelines`, `admin_*` 등
    **모든 테넌트 데이터**를 조회·관리
  - 멀티테넌트 구조 전체를 보는 “슈퍼 유저”이므로,
    모든 요청에 대해 Audit Log를 남기는 것이 필수

#### 5.5.2 HQ Admin 권한 모델

HQ Admin에서 사용하는 내부 Role 예시:

- `platform_admin` (또는 `NEURALTWIN_ADMIN`)
- `ops_admin`
- `read_only_admin`

특징:

- **고객 테넌트의 일반 Role** (`ORG_OWNER`, `HQ_ANALYST` 등)과는 별도의 레벨
- RLS에서는
  - 일반 유저: 자신의 org 데이터만
  - HQ Admin: 모든 org 데이터 + 내부 admin 테이블(`admin_*`) 접근 허용
- 정책 구현:
  - Supabase의 `auth.jwt()` 또는 별도 RBAC 테이블에서
    `is_internal = true`, `admin_role` 등을 확인하여 분기
  - Edge Functions에서도 JWT Claims 검사로 동일 정책 적용

#### 5.5.3 데이터 접근 원칙

- 고객 테넌트 데이터는 **HQ Admin에 의해 조회 가능**하되,
  - 누가 / 언제 / 어떤 테넌트/스토어/데이터를 조회했는지  
    `admin_actions` / `admin_audit_logs` 같은 테이블에 기록
- Impersonation 기능 사용 시
  - 실제 조작 권한(쓰기/삭제)은 제한하거나 별도 경고 표시
  - 우선은 “읽기 전용” 미러링 모드를 기본으로 도입

#### 5.5.4 Graph/Ontology 접근 정책

- Graph 구조(`graph_entities`, `graph_relations`)는
  **기본적으로 HQ Admin 콘솔에서만 조회/관리 가능**
  - 고객 앱에서는 직접 노드/엣지를 볼 수 없음
- 고객 앱이 Graph를 간접적으로 쓰는 경우
  - 예: 디지털 트윈 분석, 추천, 동선 분석 등
  - Graph 자체는 노출하지 않고, **결과/지표/추천**만 제공
- Ontology 스키마 변경:
  - HQ Admin에서 설계/검토 후
  - 영향 받을 테넌트/데이터 유형을 점검하고
  - 제한된 범위에서 먼저 테스트 → 전체 롤아웃

---

### 5.6 한 줄 정리

> Supabase 프로젝트 1개를  
> Auth(User) + 멀티테넌트 DB + Edge Functions의 **공통 백엔드**로 만든다.  
>  
> 3개의 Lovable 프로젝트(Website / Customer / HQ)는  
> 같은 Supabase URL/Key를 사용하면서,  
> 각자 **자기 역할에 맞는 테이블/함수**만 사용한다.  
>  
> 멀티테넌시는 `organizations + organization_members + org_id + RLS` 구조로 구현하고,  
> HQ Admin은 그 위에 **전 테넌트 관점의 권한/모니터링/거버넌스 레이어**를 올린다.


## 6. 데이터/로그 구조 개요 (Admin용 메타 데이터)

아래는 관리자 콘솔을 위해 추가될 수 있는 예시적인 메타 데이터 구조(고수준)이다.

- `admin_incidents`
  - 고객 문의/이슈 티켓과 시스템 상태를 연결
- `admin_actions`
  - 관리자 수행 행위 (재집계, 배포, 강제 제한 설정 등) Audit 로그
- `admin_scene_projects`
  - 3D 디지털 트윈 제작 요청 메타데이터
- `admin_scene_project_messages`
  - 3D 제작 요청에 대한 관리자↔사용자 메시지 히스토리
- `admin_feature_flags`
  - 기능 플래그/실험 설정 정보

구체적인 스키마는 향후 구현 단계에서 세부 설계한다.

---

## 7. 향후 확장 포인트

- 에러/데이터 이슈 발생 시 Slack/이메일 등으로 자동 알림
- 특정 패턴(예: 연속 Import 실패, 시뮬레이션 실패 급증)에 대한 룰 기반 자동 대응
- 관리자 콘솔 내 검색 기능 (테넌트/스토어/유저/이슈를 한 번에 찾는 Global Search)
- Admin용 KPI 대시보드 (전체 테넌트 활성도, 이슈 처리 SLA 등)
- 3D 디지털 트윈 제작 리드 타임/품질 지표 관리 대시보드
- Ontology/Graph 변경에 대한 자동 영향 분석 리포트


## 8. 채널별 프로젝트 가이드 (Website / Customer Dashboard / HQ Admin)

> 이 섹션은 각 채널별로 **별도 Lovable 프로젝트**를 만들 때  
> “프로젝트 설명/컨텍스트”로 그대로 붙여 넣을 수 있는 형태의 가이드이다.  
> 코어 Auth/Org/License/데이터 레이어는 세 프로젝트가 **공유**한다.

---

### 8.1 웹사이트 프로젝트 가이드 (Public Website)

#### 8.1.1 프로젝트 목적

NEURALTWIN의 마케팅/세일즈/온보딩 채널.

주요 기능:

- 서비스 소개, 기능/사례/가격
- 회원가입/로그인
- 구독 결제(플랜·점포 수 선택)
- 가입 후 **조직(브랜드) + 라이선스** 생성 및 고객 대시보드 연결

#### 8.1.2 공통 백엔드 연동 규칙

- Auth / Org / License 백엔드는 **고객 대시보드·HQ와 동일 백엔드** 사용
- 예시 환경변수:
  - `SUPABASE_URL=...`
  - `SUPABASE_ANON_KEY=...`
- Website에서 생성된 계정은 **Customer Dashboard, HQ Admin에서 그대로 로그인 가능**

회원가입 후 수행해야 하는 4단계:

1. `auth`에 사용자 생성
2. `organizations` 테이블에 org 레코드 생성
3. `organization_members`에 `(user_id, org_id, role="ORG_OWNER")` 추가
4. 구독 완료 시 `subscriptions` / `licenses`에 레코드 추가

→ 이 4단계를 **웹사이트 프로젝트가 책임지고**,  
나머지는 고객 대시보드/관리자 대시보드에서 동일 Org/License를 읽어 쓰는 구조.

#### 8.1.3 도메인/라우팅 구조

- 예시 도메인: `https://www.neuraltwin.ai`

추천 라우트:

- `/` – 랜딩 (히어로 + 주요 가치)
- `/product` – 기능 소개(Overview/Analysis/Simulation/Data Management 요약)
- `/pricing` – 플랜/요금/좌석 수 선택
- `/signup` – 회원가입
- `/login` – 로그인
- `/checkout` – 플랜 선택 후 결제
- `/demo` – 미니 피처/데모 링크
- `/legal/*` – 이용약관/개인정보처리방침 등

#### 8.1.4 구현해야 할 핵심 기능

**A. 회원가입/로그인**

- Auth Provider(Supabase/Auth0 등)와 연동
- 이메일/비밀번호 + OAuth(구글/카카오/네이버) 지원

Flow:

1. `/signup`에서 이메일/패스워드 입력 → Auth API 호출 → 유저 생성
2. 성공 시 조직 생성 API 호출:

   ```http
   POST /public/organizations
   body: { user_id, org_name, industry, country }
   응답: { org_id, default_plan }
   ```

3. `organization_members`에 `(user_id, org_id, role="ORG_OWNER")` 생성
4. 로그인 상태 유지 (JWT/세션)

**B. 구독 결제 (플랜 + 점포 수)**

- `/pricing` 페이지:
  - Starter / Pro / Enterprise 플랜 카드
  - 매장 수, HQ seat 수 슬라이더/입력
  - [시작하기] → `/checkout`
- `/checkout`:
  - 결제 수단/카드 정보 입력 후 PG/Stripe Checkout
- 결제 완료 후 PG Webhook에서:
  - `subscriptions` 테이블 업데이트
  - `licenses` 테이블에 `org_id`, `store_quota`, `hq_seat_quota` 기록

**C. 가입 후 고객 대시보드로 연결**

- 결제/가입 후 마무리 페이지:

  > “A매장(강남) 설정을 시작하세요 → [대시보드로 이동]”

- 버튼: `https://app.neuraltwin.ai/login?org_id=...` 로 리다이렉트

#### 8.1.5 Lovable용 요약 프롬프트 예시

> 이 프로젝트는 NEURALTWIN 제품의 Public Website입니다.  
>  
> 목표:
> - Supabase(Auth)와 같은 백엔드를 사용하여 회원가입/로그인을 처리합니다.
> - 가입이 완료되면 organizations / organization_members / subscriptions / licenses 테이블에 기록합니다.
> - 같은 Auth/Org를 사용하는 Customer Dashboard와 연동됩니다.  
>  
> 필수 페이지:
> - /, /product, /pricing, /signup, /login, /checkout, /demo, /legal/*  
>  
> 주의:
> - Auth/DB URL/Key는 Customer Dashboard/HQ Admin과 동일하게 설정합니다.
> - 회원가입 및 결제 완료 후, org_id/plan/seat 정보를 백엔드에 저장하고 /app으로 리다이렉트합니다.

---

### 8.2 고객 대시보드 프로젝트 가이드 (Customer Dashboard)

#### 8.2.1 프로젝트 목적

NEURALTWIN의 실제 사용자용 **리테일 OS**.

주요 기능:

- 매장/데이터/퍼널 분석 (**Analysis**)
- 레이아웃/수요/재고/가격/프로모션 시뮬레이션 (**Simulation**)
- 디지털 트윈 관리·데이터 임포트 (**Data Management**)
- 플랜/조직 기본 설정 (**Overview**)

#### 8.2.2 공통 백엔드 연동

- Website와 **같은 Auth/Org/License 백엔드** 사용
- 로그인 시:
  - `user_id + org_id`를 프런트 컨텍스트에 보관
  - 모든 API 호출에 `org_id` 포함, 또는 서버/BFF에서 JWT에서 추출

예: Supabase 사용 시

```ts
const { data: session } = await supabase.auth.getSession();
const orgId = session?.user?.user_metadata?.org_id;
```

#### 8.2.3 IA / 라우트 구조 (고객용)

1. **Overview**

   - `/dashboard`
   - `/stores`
   - `/hq-store-sync`
   - `/settings`

2. **Analysis**

   - `/analysis/footfall`
   - `/analysis/traffic-heatmap`
   - `/analysis/customer-journey`
   - `/analysis/conversion-funnel`
   - `/analysis/customer-analysis`
   - `/analysis/inventory`
   - `/analysis/profit-center`
   - `/analysis/product-performance`

3. **Simulation**

   - `/simulation/hub`
   - `/simulation/layout`
   - `/simulation/demand`
   - `/simulation/inventory`
   - `/simulation/pricing`
   - `/simulation/recommendation`

4. **Data Management (테넌트 범위)**

   - `/digital-twin-3d`
   - `/data-import`
   - `/schema-builder` (제한된 기능)

#### 8.2.4 핵심 기능별 가이드

**A. `/dashboard` (Overview)**

- 입력: `org_id`, store 필터, 기간 필터
- 데이터 소스:
  - `dashboard_kpis`, `ai_recommendations`, `stores`
- 표시:
  - KPI 카드, 퍼널 요약, 상위/하위 매장
  - “오늘의 AI 추천 액션” (Simulation Hub 결과 반영)

**B. Analysis 섹션**

각 페이지 공통:

- Filter Bar: Store, Date Range, Segment 등
- 중앙: 차트/히트맵/퍼널
- 우측/하단: TopN/인사이트 카드

Backend 예:

```http
GET /app/analysis/footfall?store_id=A001&date_from=...
```

**C. Simulation 섹션**

핵심: `/simulation/hub` + 모듈별 페이지를 **같은 패턴**으로 구현.

- Hub:
  - 5개 모듈 카드
  - [시뮬레이션 실행], [전체 분석 실행]
- 각 모듈:
  - 파라미터 폼 → `advanced-ai-inference` Edge Function 호출
  - 결과 수치 + 인사이트 + Action 저장

레이아웃/수요/재고/가격/프로모션에 대한 시뮬레이션 허브 상세 가이드는  
별도 문서/스펙에 따라 구현.

**D. Data Management 섹션**

- `/digital-twin-3d`:
  - `store_scenes`, `furniture_layout`, `product_placement` 등을 조회/저장
  - 고객 테넌트가 **자신의 매장**을 직접 편집 가능
- `/data-import`:
  - POS/CRM/ERP/센서 데이터의 CSV/API 등록 UI
  - 업로드 → `user_data_imports` 기록 → 후속 ETL/Edge Function 트리거

#### 8.2.5 Lovable용 요약 프롬프트 예시

> 이 프로젝트는 NEURALTWIN의 Customer Dashboard(고객용 리테일 OS)입니다.  
>  
> 전제:
> - Auth/Org/License 백엔드는 Website/HQ Admin과 동일한 Supabase 프로젝트를 사용합니다.
> - 로그인 후, user는 반드시 org_id 컨텍스트로 동작합니다.
> - 모든 화면은 4개의 섹션으로 구성됩니다: Overview / Analysis / Simulation / Data Management.  
>  
> 필수 구현:
> - /dashboard: 주요 KPI/퍼널/AI 추천 액션 카드
> - /analysis/*: footfall, heatmap, journey, funnel, product performance 등
> - /simulation/hub 및 /simulation/*: layout, demand, inventory, pricing, promotion 시뮬레이션
> - /digital-twin-3d: 3D Twin 뷰어/에디터 사용  
>  
> Simulation Hub:
> - /simulation/hub에서 5개의 모듈(Layout/Demand/Inventory/Pricing/Promotion)을 카드 형태로 노출합니다.
> - 각 모듈은 scenarioType과 params를 받아 advanced-ai-inference Edge Function을 호출합니다.
> - 결과로 반환되는 ΔKPI와 인사이트를 카드/차트에 보여줍니다.

---

### 8.3 관리자 대시보드 프로젝트 가이드 (NEURALTWIN HQ)

#### 8.3.1 프로젝트 목적

NEURALTWIN 내부 팀(HQ)을 위한 **운영 콘솔**.

주요 기능:

- 전체 Tenants / Organizations / Subscriptions / License 관리
- 온톨로지 스키마 / Ontology Version 관리
- 데이터 소스 / ETL 파이프라인 / 에러 모니터링
- AI 모델 / Simulation Config 관리
- Audit / 보안 / 권한 관리

#### 8.3.2 Auth / 권한

- Website/Customer와 **같은 Auth 백엔드** 사용
- 단, **내부 계정만 접근 가능**

역할 예시:

- `NEURALTWIN_ADMIN`
- `INTERNAL_USER`

정책:

- JWT claims 또는 RBAC 테이블에서 `is_internal = true` 플래그 체크
- 내부 도메인 예: `https://hq.neuraltwin.ai`

#### 8.3.3 IA / 라우트 구조 (HQ용 요약)

1. **Tenants / Billing**
   - `/hq/tenants`
   - `/hq/licenses`
   - `/hq/subscriptions`
   - `/hq/usage`

2. **Data & Ontology**
   - `/hq/ontology-schema`
   - `/hq/data-sources`
   - `/hq/pipelines`

3. **Simulation & Models**
   - `/hq/simulation-config`
   - `/hq/model-versions`
   - `/hq/experiments` (A/B 테스트)

4. **Ops / Security**
   - `/hq/users`
   - `/hq/audit-logs`
   - `/hq/alerts`

> 본 문서의 2–3장에 나오는 상세 IA/섹션 설계는  
> 위 라우트 구조를 기준으로 확장·구체화한 것이다.

#### 8.3.4 기능별 상세 요약

**A. `/hq/tenants` & `/hq/licenses`**

- 리스트:
  - `org_id`, `org_name`, `plan`, `status`, `store_count`, `created_at`
- 하단:
  - 라이선스 (점포 수, HQ seats, 만료일 등)
- 액션:
  - 테넌트 생성/비활성화
  - 플랜/쿼터 변경
  - 테스트/샌드박스 테넌트 생성

**B. `/hq/ontology-schema`**

- 온톨로지 엔티티/관계 타입 편집 UI
  - `entity_type_id`, `display_name`, 필수 속성 목록, 설명
- 각 변경 사항에 대해:
  - 버전 관리
  - 주요 테넌트에 대한 영향도 표시 (ex. schema v1 → v2)

**C. `/hq/data-sources` & `/hq/pipelines`**

- 테넌트별 데이터 소스(연결된 POS/CRM/ERP/센서) 목록
- ETL Job 상태:
  - success/fail, last run, duration
- ETL 재시작/강제 실행 버튼
- 에러 로그/누락 데이터 탐색

**D. `/hq/simulation-config` & `/hq/model-versions`**

- 기본 시뮬레이션 파라미터 관리:
  - 예: layout_optimization에서 ΔCVR 상/하한, 수요/가격 탄력성 파라미터
- 모델 버전:
  - active version, candidate version, rollback 버전 관리
- A/B 실험 (`/hq/experiments`):
  - 일부 테넌트에만 새 버전 적용, 결과 모니터링

**E. `/hq/audit-logs` & `/hq/alerts`**

- Audit:
  - HQ 콘솔에서 일어난 중요한 변경 (플랜 변경, 스키마 변경 등) 기록
- Alerts:
  - ETL 실패, AI inference 오류, 데이터 지연, 월별 usage 폭증 등

#### 8.3.5 Lovable용 요약 프롬프트 예시

> 이 프로젝트는 NEURALTWIN 팀 내부용 HQ Admin Dashboard입니다.  
>  
> 역할:
> - Tenants(고객 조직) / Licenses / Subscriptions 관리
> - Ontology Schema/Version 관리
> - 데이터 소스/ETL 파이프라인 모니터링
> - 시뮬레이션/AI 모델 버전 관리
> - 감사로그/알림/보안 관제  
>  
> Auth:
> - Website/Customer와 동일한 Auth 백엔드를 사용하지만,
> - 로그인 사용자는 role이 NEURALTWIN_ADMIN 또는 is_internal=true 여야 합니다.  
>  
> 필수 페이지:
> - /hq/tenants, /hq/licenses, /hq/subscriptions, /hq/usage
> - /hq/ontology-schema, /hq/data-sources, /hq/pipelines
> - /hq/simulation-config, /hq/model-versions, /hq/experiments
> - /hq/users, /hq/audit-logs, /hq/alerts  
>  
> 주의:
> - 이 프로젝트에서는 multi-tenant 데이터를 전부 볼 수 있으므로,
> - 모든 API 호출에 대해 권한/로그를 반드시 남겨야 합니다.
