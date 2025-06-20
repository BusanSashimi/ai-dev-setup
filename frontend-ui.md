# 프론트엔드 UI 자동 생성 가이드

## 🎯 목적

**기존에 생성된 Store와 Type을 기반으로** Vue3 UI 컴포넌트를 자동 생성합니다.

- 실제 동작하는 완전한 CRUD 기능 구현
- 백엔드 API와 완전 연동된 UI 구현
- 프로젝트 표준 규칙을 완벽히 준수하는 컴포넌트 생성

## 🔧 사용 방법

**기본 명령어:**

```bash
/run [모듈명]                    # 기본 CRUD 페이지 생성
/run [모듈명] two                # 투뎁스(Two-Depth) 레이아웃 생성
/run [모듈명] select             # 선택 모달(Select Modal) 생성
```

**📝 모듈명 자동 변환:**

언더스코어(`_`)를 포함한 모듈명은 자동으로 케밥 케이스(`-`)로 변환됩니다:

```bash
/run member_auth                 # → member-auth 기본 CRUD UI 생성
/run member_auth two             # → member-auth 투뎁스 레이아웃 UI 생성
/run member_auth select          # → member-auth 선택 모달 UI 생성
/run itax_client                 # → itax-client 기본 CRUD UI 생성
/run itax_client two             # → itax-client 투뎁스 레이아웃 UI 생성
/run itax_client select          # → itax-client 선택 모달 UI 생성
```

**📋 옵션별 생성되는 컴포넌트:**

- **기본 (`/run [모듈명]`)**:
  - `list.vue`, `list-search.vue`, `list-table.vue` (목록 페이지)
  - `detail.vue` (상세 페이지)
  - `insert.vue`, `update.vue` (등록/수정 페이지)

- **투뎁스 (`/run [모듈명] two`)**:
  - `list.vue` (좌측: 검색+목록, 우측: 상세뷰)
  - `list-search.vue`, `list-table.vue` (좌측 패널용)
  - `list-detail.vue` (우측 상세 패널, 탭 구조 포함)
  - `insert.vue`, `update.vue` (모달 방식으로 기존과 동일)

- **선택 모달 (`/run [모듈명] select`)**:
  - `list-table-select.vue` (선택 모달 컴포넌트)
  - `demo.vue` (선택 모달 사용 예제 페이지)
  - **주의**: 등록/수정/삭제 기능은 생성되지 않음 (선택 전용)

## 📋 프로젝트 핵심 규칙

### 파일 명명 규칙

- 컴포넌트: **케밥 케이스(kebab-case)** 사용
- 모달 컴포넌트: `modal-` 접두사 사용
- 리스트: `list.vue`, `list-search.vue`, `list-table.vue`
- 상세: `modal-detail.vue` 또는 `detail.vue`
- 등록: `modal-insert.vue` 또는 `insert.vue`
- 수정: `modal-update.vue` 또는 `update.vue`
- **Validator**: `[모듈명].validator.ts`
- **라우터**: `[모듈명].routes.ts`

### Vue3 개발 규칙

- **Composition API 방식**으로 코딩 (`<script setup>` 사용)
- **Store 활용**: `const store = use[ModuleName]Store()` 패턴
- 모든 UI는 최신 스타일로 작성 (NuxtUI v3, TailwindCSS v4)
- 모바일 & 데스크톱 반응형 디자인 필수

### Store 연동 규칙

- **상태 활용**: Store의 `listData`, `detailData`, `listParams` 활용
- **함수 호출**: Store의 API 함수들을 컴포넌트에서 직접 호출
- **반응성**: Store 상태 변경 시 UI 자동 업데이트
- **초기화**: 컴포넌트 마운트 시 필요한 Store 함수 호출

### 컴포넌트 구조 및 기준 코드

**기본 CRUD 패턴 (`/run [모듈명]`)**:
- **기준 폴더**: `webapp/src/modules/test/pages/crud/` 폴더의 구조를 **반드시** 그대로 따름
- **`list.vue`**: `list-search.vue`와 `list-table.vue`를 포함하는 껍데기(shell) 컴포넌트
- **`list-search.vue`**: `SearchDto`를 기반으로 한 모든 검색 UI 요소 포함
- **`list-table.vue`**: 목록을 표시하는 `EasyDataTable`과 CRUD 버튼 포함

**투뎁스 레이아웃 패턴 (`/run [모듈명] two`)**:
- **기준 폴더**: `webapp/src/modules/test/pages/two-depth/` 폴더의 구조를 **반드시** 그대로 따름
- **`list.vue`**: 좌측에 `list-search.vue`와 `list-table.vue`, 우측에 `list-detail.vue`를 배치한 투뎁스 레이아웃
- **`list-search.vue`**: 좌측 패널용 검색 폼 (기본 패턴과 동일)
- **`list-table.vue`**: 좌측 패널용 목록 테이블 (선택 시 우측 상세뷰 업데이트)
- **`list-detail.vue`**: 우측 패널용 상세뷰 (탭 구조 포함, 선택된 항목의 상세 정보 표시)

**선택 모달 패턴 (`/run [모듈명] select`)**:
- **기준 폴더**: `webapp/src/modules/test/pages/select-list/` 폴더의 구조를 **반드시** 그대로 따름
- **`list-table-select.vue`**: 모달 기반 선택 컴포넌트 (검색, 목록, 체크박스 선택 기능)
- **`demo.vue`**: 선택 모달 사용 예제 및 테스트 페이지
- **특징**: 단일 선택 및 다중 선택 지원, 선택된 항목 badge 표시

**공통 UI 컴포넌트**:
- **UFormField**: 상세 보기에 사용
- **UModal**: 모달 화면에 사용 (등록/수정/선택)
- **USwitch**: '여부' 필드에 사용
- **UTabs**: 투뎁스 패턴의 상세뷰 탭에 사용
- **UBadge**: 선택 모달의 선택된 항목 표시에 사용

### TailwindCSS 규칙

- **5개 이상의 클래스 시 배열로 그룹핑**
- 클래스 그룹화 순서:
  1. Layout, Flexbox, Grid, Spacing, Sizing
  2. Typography, Background, Borders
  3. Pseudo-classes, Media Queries, Hover/Focus
  4. Filters, Transitions, Animations, Transforms

## 🤖 AI 수행 작업

AI는 다음 순서에 따라 UI 컴포넌트와 관련 파일들을 생성합니다. 각 단계는 조건부로 실행될 수 있습니다.

### 1단계: 사전 분석 및 생성 계획 수립

가장 먼저, 기존 Store와 Type 파일을 분석하고, 옵션에 따라 어떤 컴포넌트를 생성할지 결정합니다.

- **분석 대상 파일:**

  - `webapp/src/modules/[모듈명]/store/[모듈명].store.ts`
  - `webapp/src/modules/[모듈명]/type/[모듈명].type.ts`

- **확인 항목:**

  - **Store 함수:** `paging`, `detail`, `insert`, `update`, `updateUse`, `softDelete` 등의 존재 여부
  - **Type 인터페이스:** `SearchDto`, `InsertDto`, `UpdateDto` 등의 구조 (특히 `SearchDto`를 중점적으로 분석)
  - **옵션 확인:** `two` 옵션 여부 확인

- **생성 계획 (기본 패턴):**
  - `paging()` 함수 존재 시 → 목록 관련 파일 3개 (`list.vue`, `list-search.vue`, `list-table.vue`) 생성 계획
  - `detail()` 함수 존재 시 → `detail.vue` 생성 계획
  - `insert()` 함수 존재 시 → `insert.vue` 생성 계획
  - `update()` 함수 존재 시 → `update.vue` 생성 계획
  - `insert()` 또는 `update()` 함수 존재 시 → `[모듈명].validator.ts` 생성 계획
  - 마지막으로, 생성된 컴포넌트를 위한 `[모듈명].routes.ts` 생성 계획

- **생성 계획 (투뎁스 패턴 - `two` 옵션 사용 시):**
  - `paging()` 함수 존재 시 → 투뎁스 레이아웃 파일 4개 (`list.vue`, `list-search.vue`, `list-table.vue`, `list-detail.vue`) 생성 계획
  - `insert()` 함수 존재 시 → `insert.vue` 생성 계획 (모달 방식)
  - `update()` 함수 존재 시 → `update.vue` 생성 계획 (모달 방식)
  - `insert()` 또는 `update()` 함수 존재 시 → `[모듈명].validator.ts` 생성 계획
  - 마지막으로, 생성된 컴포넌트를 위한 `[모듈명].routes.ts` 생성 계획
  - **주의:** 투뎁스 패턴에서는 별도의 `detail.vue` 페이지를 생성하지 않음 (`list-detail.vue`가 대체)

- **생성 계획 (선택 모달 패턴 - `select` 옵션 사용 시):**
  - `paging()` 함수 존재 시 → 선택 모달 파일 2개 (`list-table-select.vue`, `demo.vue`) 생성 계획
  - 마지막으로, 생성된 컴포넌트를 위한 `[모듈명].routes.ts` 생성 계획
  - **주의:** 선택 모달 패턴에서는 `insert()`, `update()`, `softDelete()` 관련 컴포넌트를 생성하지 않음 (선택 전용)

### 2단계: UI 페이지 및 Validator 파일 생성 (조건부)

사전 분석 결과를 바탕으로, `webapp/src/modules/[모듈명]/pages/` 경로에 필요한 파일을 생성합니다.

#### 기본 패턴 컴포넌트 생성

- **기준 코드:** 모든 파일은 `webapp/src/modules/test/pages/crud/` 폴더의 파일을 기준으로 작성합니다.

1.  **`list.vue`, `list-search.vue`, `list-table.vue` 생성 (`paging()` 함수 존재 시)**

    - **`list-search.vue`**:
      - Store의 `listParams`와 `SearchDto`를 연동하여 모든 검색 UI 요소를 구현합니다.
      - '검색' 버튼은 Store의 `paging()` 함수를, '초기화' 버튼은 `listParamsInit()` 함수를 호출합니다.
    - **`list-table.vue`**:
      - `EasyDataTable`을 사용하여 Store의 `listData`와 `listTotalRow`를 연동합니다.
      - 페이지네이션 및 정렬 옵션을 Store의 `listParams`와 연동합니다.
      - Store 함수 존재 여부에 따라 '등록', '수정', '삭제', '사용여부' 등의 기능을 버튼 및 스위치로 구현합니다.
    - **`list.vue`**:
      - 위에서 생성한 `list-search.vue`와 `list-table.vue`를 포함하는 껍데기(shell) 컴포넌트입니다.

2.  **`detail.vue` 생성 (`detail()` 함수 존재 시)**

    - Store의 `detailData` 상태를 연동합니다.
    - `UFormField`를 사용하여 상세 데이터를 표시합니다.

#### 투뎁스 패턴 컴포넌트 생성 (`two` 옵션 사용 시)

- **기준 코드:** 모든 파일은 `webapp/src/modules/test/pages/two-depth/` 폴더의 파일을 기준으로 작성합니다.

1.  **`list.vue` 생성 (투뎁스 레이아웃)**

    - 좌측: `list-search.vue`와 `list-table.vue`를 포함하는 좌측 패널 (고정 최대 너비 350px)
    - 우측: `list-detail.vue`를 포함하는 우측 상세 패널 (가변 너비)
    - Flexbox 레이아웃을 사용하여 두 패널을 수평 배치합니다.

2.  **`list-search.vue`, `list-table.vue` 생성 (좌측 패널용)**

    - **`list-search.vue`**: 기본 패턴과 동일하게 검색 폼 구현
    - **`list-table.vue`**: 기본 패턴과 유사하지만, 행 클릭 시 우측 상세뷰를 업데이트하는 기능 추가
      - 행 클릭 시 `router.push({ query: { ...route.query, selected: item.id } })` 방식으로 선택 상태 관리

3.  **`list-detail.vue` 생성 (우측 패널용)**

    - 선택된 항목이 없을 때: "데이터를 선택해주세요." 메시지 표시
    - 선택된 항목이 있을 때: 상세 정보를 탭 구조로 표시
    - `UTabs` 컴포넌트를 사용하여 여러 탭 구성 (기본정보, 추가정보 등)
    - 수정/삭제 버튼 포함 (모달 방식)
    - `route.query.selected` 값을 감시하여 자동으로 상세 데이터 조회

#### 선택 모달 패턴 컴포넌트 생성 (`select` 옵션 사용 시)

- **기준 코드:** 모든 파일은 `webapp/src/modules/test/pages/select-list/` 폴더의 파일을 기준으로 작성합니다.

1.  **`list-table-select.vue` 생성 (`paging()` 함수 존재 시)**

    - 모달 기반 선택 컴포넌트 구현
    - 상단: 키워드 검색 폼 (Store의 `listParams`와 연동)
    - 중앙: `EasyDataTable`을 사용한 목록 표시 (체크박스 선택 지원)
    - 하단: 선택된 항목을 Badge로 표시하는 영역
    - 단일 선택(`oneSelect`) 및 다중 선택(`checkSelect`) 기능 지원
    - Props: `open` (모달 열림/닫힘), `selectList` (기존 선택 목록)
    - Emits: `update:open`, `close`, `select-ok`

2.  **`demo.vue` 생성**

    - 선택 모달 사용 예제 및 테스트 페이지
    - 선택 모달을 호출하는 버튼과 선택된 데이터를 표시하는 영역 구현
    - 선택된 데이터의 JSON 형태 및 통계 정보 표시
    - 선택 초기화 기능 포함

#### 공통 컴포넌트 생성 (기본/투뎁스 패턴만 해당)

1.  **`insert.vue` 생성 (`insert()` 함수 존재 시)**

    - `[ModuleName]InsertDto` 타입을 활용하여 입력 폼을 구성합니다.
    - '저장' 버튼 클릭 시 Store의 `insert()` 함수를 호출합니다.
    - 아래에서 생성할 Validator를 적용하여 유효성 검사를 수행합니다.

2.  **`update.vue` 생성 (`update()` 함수 존재 시)**

    - `[ModuleName]UpdateDto` 타입을 활용하여 수정 폼을 구성합니다.
    - `onMounted` 시 Store의 `detail()` 함수를 호출하여 초기 데이터를 가져옵니다.
    - '수정' 버튼 클릭 시 Store의 `update()` 함수를 호출합니다.
    - Validator를 적용하여 유효성 검사를 수행합니다.

3.  **`[모듈명].validator.ts` 생성 (`insert()` 또는 `update()` 함수 존재 시)**
    - 기본 패턴: `webapp/src/modules/test/pages/crud/_crud.validator.ts` 파일을 참고
    - 투뎁스 패턴: 동일한 Validator 사용 (패턴에 상관없이 동일)
    - 선택 모달 패턴: Validator 생성하지 않음 (등록/수정 기능 없음)
    - `yup` 라이브러리를 사용하여 `[ModuleName]InsertValidator`와 `[ModuleName]UpdateValidator` 스키마를 정의합니다.
    - 에러 메시지는 **한국어**로 작성합니다.

### 3단계: 라우터 설정

1.  **`[모듈명].routes.ts` 파일 생성**

    - `webapp/src/modules/[모듈명]/[모듈명].routes.ts` 경로에 생성합니다.
    - **실제 생성된 컴포넌트만** 라우트 설정에 포함시킵니다.
    - ❗️상세, 등록, 수정 화면은 모달(Modal)로 구현되므로, 일반적으로 `list.vue` 페이지만 라우트로 등록됩니다.

2.  **메인 라우터에 추가**
    - `webapp/src/router.ts` 파일에 생성된 라우트 설정을 추가합니다.

### 4단계: 최종 품질 검증

모든 파일 생성이 완료된 후, 아래 절차에 따라 코드 품질을 검증합니다.

- **Store 연동 검증**: 존재하는 Store 함수가 UI에서 정상 호출되는지 확인
- **SearchDto 활용 검증**: 검색 기능이 SearchDto 구조와 완벽히 매핑되는지 확인
- **Type 활용 검증**: 모든 Type이 UI에서 정확히 활용되는지 확인
- **CRUD 기능 검증**: 실제 백엔드와 연동된 모든 기능 테스트
- **린트 검사**: `npm run lint:fix` 실행
- **타입 체크**: `npx vue-tsc --noEmit` 실행
- **빌드 테스트**: `npm run build` 실행하여 최종 검증

## ✅ 완료 확인

### 생성된 파일 체크리스트 (조건부)

#### 기본 패턴 (`/run [모듈명]`)

**⚠️ Store에 해당 함수가 존재할 때만 생성되는 컴포넌트:**

- [ ] `webapp/src/modules/[모듈명]/pages/list.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/list-search.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/list-table.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/detail.vue` (`detail()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/insert.vue` (`insert()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/update.vue` (`update()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/[모듈명].validator.ts` (`insert()` 또는 `update()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/[모듈명].routes.ts` (생성된 컴포넌트 기준)

#### 투뎁스 패턴 (`/run [모듈명] two`)

**⚠️ Store에 해당 함수가 존재할 때만 생성되는 컴포넌트:**

- [ ] `webapp/src/modules/[모듈명]/pages/list.vue` (`paging()` 함수 존재 시 - 투뎁스 레이아웃)
- [ ] `webapp/src/modules/[모듈명]/pages/list-search.vue` (`paging()` 함수 존재 시 - 좌측 패널용)
- [ ] `webapp/src/modules/[모듈명]/pages/list-table.vue` (`paging()` 함수 존재 시 - 좌측 패널용)
- [ ] `webapp/src/modules/[모듈명]/pages/list-detail.vue` (`paging()` 함수 존재 시 - 우측 패널용)
- [ ] `webapp/src/modules/[모듈명]/pages/insert.vue` (`insert()` 함수 존재 시 - 모달)
- [ ] `webapp/src/modules/[모듈명]/pages/update.vue` (`update()` 함수 존재 시 - 모달)
- [ ] `webapp/src/modules/[모듈명]/pages/[모듈명].validator.ts` (`insert()` 또는 `update()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/[모듈명].routes.ts` (생성된 컴포넌트 기준)

**📋 투뎁스 패턴 특징:**
- `detail.vue` 페이지는 생성되지 않음 (`list-detail.vue`가 대체)
- `list.vue`는 좌우 분할 레이아웃 구조
- `list-detail.vue`는 탭 구조로 상세 정보 표시
- 등록/수정은 모달 방식으로 동일하게 동작

#### 선택 모달 패턴 (`/run [모듈명] select`)

**⚠️ Store에 해당 함수가 존재할 때만 생성되는 컴포넌트:**

- [ ] `webapp/src/modules/[모듈명]/pages/list-table-select.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/demo.vue` (선택 모달 테스트 페이지)
- [ ] `webapp/src/modules/[모듈명]/[모듈명].routes.ts` (생성된 컴포넌트 기준)

**📋 선택 모달 패턴 특징:**
- 등록/수정/삭제 기능은 생성되지 않음 (선택 전용)
- 모달 기반 선택 인터페이스
- 단일 선택 및 다중 선택 지원
- 선택된 항목을 Badge로 시각적 표시

### Store 함수 존재 여부 확인

**먼저 Store 파일에서 다음 함수들의 존재 여부를 확인:**

- [ ] `paging()` 함수 존재 여부 확인
- [ ] `list()` 함수 존재 여부 확인 (검색 기능용)
- [ ] `detail()` 함수 존재 여부 확인
- [ ] `insert()` 함수 존재 여부 확인
- [ ] `update()` 함수 존재 여부 확인
- [ ] `updateUse()` 함수 존재 여부 확인
- [ ] `softDelete()` 함수 존재 여부 확인

### 품질 검증 단계

1. **파일 생성 완료 확인**

   ```bash
   ls -la webapp/src/modules/[모듈명]/pages/
   ```

2. **타입 체크 및 린트 검사**

   ```bash
   npx vue-tsc --noEmit
   npm run lint:fix
   ```

3. **빌드 테스트**

   ```bash
   npm run build
   ```

### 최종 품질 체크리스트

- [ ] UI 컴포넌트가 정상 생성됨
- [ ] 모든 Store 함수가 UI에서 정상 호출됨
- [ ] 모든 Type이 UI에서 정확히 활용됨
- [ ] SearchDto가 완전히 활용됨 (모든 검색 필드 구현)
- [ ] 실제 백엔드 API와 완전 연동 확인
- [ ] CRUD 모든 기능이 정상 동작함
- [ ] 검색, 페이징, 정렬 기능 정상 동작함
- [ ] 유효성 검사가 정상 적용됨 (insert/update 시)
- [ ] 반응형 디자인이 정상 동작함
- [ ] 타입 에러 없음 (`vue-tsc` 통과)
- [ ] 린트 에러 없음 (`npm run lint:fix` 통과)
- [ ] 빌드 에러 없음 (`npm run build` 통과)
- [ ] UI/UX가 test/crud 폴더와 동일한 수준

## 💡 중요 사항

### 🚨 반드시 지켜야 할 3가지 핵심 규칙

1. **Store 완전 활용**: 기존 Store의 모든 함수와 상태를 빠짐없이 UI에서 활용
2. **Type 정확 매핑**: 기존 Type 인터페이스를 정확히 활용한 데이터 바인딩
3. **실제 기능 구현**: 목업이 아닌 실제 백엔드 API와 연동된 완전한 기능 구현

### 📋 개발 지침

#### 기본 패턴
- **test/crud 폴더 기준**: 모든 구조와 디자인은 `webapp/src/modules/test/pages/crud/` 기준
- **개별 페이지 구조**: 목록, 상세, 등록, 수정 페이지가 각각 독립적으로 구성
- **라우터 기반 네비게이션**: 페이지 간 이동은 라우터 기반

#### 투뎁스 패턴 (`two` 옵션)
- **test/two-depth 폴더 기준**: 모든 구조와 디자인은 `webapp/src/modules/test/pages/two-depth/` 기준
- **마스터-디테일 구조**: 좌측 목록, 우측 상세뷰의 분할 레이아웃
- **쿼리 기반 선택**: `route.query.selected`로 선택된 항목 관리
- **탭 기반 상세뷰**: 우측 패널에서 탭 구조로 상세 정보 표시
- **모달 등록/수정**: 등록/수정은 기본 패턴과 동일하게 모달 방식

#### 선택 모달 패턴 (`select` 옵션)
- **test/select-list 폴더 기준**: 모든 구조와 디자인은 `webapp/src/modules/test/pages/select-list/` 기준
- **모달 기반 선택**: 데이터 선택을 위한 모달 인터페이스
- **검색 및 페이지네이션**: 키워드 검색과 페이지네이션 지원
- **다중/단일 선택**: 체크박스를 통한 다중 선택 및 개별 버튼을 통한 단일 선택
- **선택 표시**: Badge 형태로 선택된 항목 시각적 표시
- **CRUD 제외**: 등록/수정/삭제 기능은 생성하지 않음

#### 공통 지침
- **Store 중심 개발**: UI는 Store의 상태와 함수를 중심으로 구현
- **조건부 컴포넌트 생성**: Store에 해당 함수가 존재할 때만 관련 컴포넌트 생성
- **SearchDto 완전 활용**: 검색 기능은 SearchDto의 모든 필드를 빠짐없이 활용
- **Type 기반 개발**: 모든 데이터 처리는 기존 Type 인터페이스 기반
- **완전한 연동**: 부분적 구현이 아닌 Store의 모든 기능을 완전 연동

## 📊 패턴별 비교표

| 구분 | 기본 패턴 | 투뎁스 패턴 (two) | 선택 모달 패턴 (select) |
|------|-----------|-------------------|-------------------------|
| **레이아웃** | 개별 페이지 구조 | 좌우 분할 구조 | 모달 기반 구조 |
| **목록 표시** | 독립된 목록 페이지 | 좌측 패널 고정 표시 | 모달 내 목록 표시 |
| **상세 보기** | 별도 상세 페이지 | 우측 패널 즉시 표시 | 없음 (선택 전용) |
| **등록/수정** | 모달 또는 페이지 | 모달 방식 | 없음 |
| **선택 기능** | 없음 | 없음 | 단일/다중 선택 지원 |
| **네비게이션** | 라우터 기반 | 쿼리 기반 선택 + 모달 | 모달 기반 |
| **사용 케이스** | 일반적인 CRUD | 빠른 조회/편집이 필요한 경우 | 데이터 선택이 필요한 경우 |
| **기준 코드** | test/pages/crud/ | test/pages/two-depth/ | test/pages/select-list/ |

- **투뎁스 패턴**: 사용자가 목록에서 항목을 선택하면 즉시 우측에 상세 정보가 표시되어, 빠른 데이터 조회 및 편집이 필요한 업무에 적합합니다.
- **선택 모달 패턴**: 다른 폼이나 페이지에서 특정 데이터를 선택해야 할 때 사용하는 모달 인터페이스로, 검색과 선택 기능에 특화되어 있습니다.
