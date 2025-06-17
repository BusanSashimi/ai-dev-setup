# 프론트엔드 UI 자동 생성 가이드

## 🎯 목적

**기존에 생성된 Store와 Type을 기반으로** Vue3 UI 컴포넌트를 자동 생성합니다.

- 실제 동작하는 완전한 CRUD 기능 구현
- 백엔드 API와 완전 연동된 UI 구현
- 프로젝트 표준 규칙을 완벽히 준수하는 컴포넌트 생성

## 🔧 사용 방법

**명령어:**

```bash
/run [모듈명]
```

**📝 모듈명 자동 변환:**

언더스코어(`_`)를 포함한 모듈명은 자동으로 케밥 케이스(`-`)로 변환됩니다:

```bash
/run member_auth         # → member-auth UI 생성
/run itax_client         # → itax-client UI 생성
```

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

- **기준 폴더**: `webapp/src/modules/test/pages/crud/` 폴더의 구조를 **반드시** 그대로 따름
- **`list.vue`**: `list-search.vue`와 `list-table.vue`를 포함하는 껍데기(shell) 컴포넌트
- **`list-search.vue`**: `SearchDto`를 기반으로 한 모든 검색 UI 요소 포함
- **`list-table.vue`**: 목록을 표시하는 `EasyDataTable`과 CRUD 버튼 포함
- **UFormField**: 상세 보기에 사용
- **UModal**: 모달 화면에 사용
- **USwitch**: '여부' 필드에 사용

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

가장 먼저, 기존 Store와 Type 파일을 분석하여 어떤 컴포넌트를 생성할지 결정합니다.

- **분석 대상 파일:**

  - `webapp/src/modules/[모듈명]/store/[모듈명].store.ts`
  - `webapp/src/modules/[모듈명]/type/[모듈명].type.ts`

- **확인 항목:**

  - **Store 함수:** `paging`, `detail`, `insert`, `update`, `updateUse`, `softDelete` 등의 존재 여부
  - **Type 인터페이스:** `SearchDto`, `InsertDto`, `UpdateDto` 등의 구조 (특히 `SearchDto`를 중점적으로 분석)

- **생성 계획:**
  - `paging()` 함수 존재 시 → 목록 관련 파일 3개 (`list.vue`, `list-search.vue`, `list-table.vue`) 생성 계획
  - `detail()` 함수 존재 시 → `detail.vue` 생성 계획
  - `insert()` 함수 존재 시 → `insert.vue` 생성 계획
  - `update()` 함수 존재 시 → `update.vue` 생성 계획
  - `insert()` 또는 `update()` 함수 존재 시 → `[모듈명].validator.ts` 생성 계획
  - 마지막으로, 생성된 컴포넌트를 위한 `[모듈명].routes.ts` 생성 계획

### 2단계: UI 페이지 및 Validator 파일 생성 (조건부)

사전 분석 결과를 바탕으로, `webapp/src/modules/[모듈명]/pages/` 경로에 필요한 파일을 생성합니다.

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

3.  **`insert.vue` 생성 (`insert()` 함수 존재 시)**

    - `[ModuleName]InsertDto` 타입을 활용하여 입력 폼을 구성합니다.
    - '저장' 버튼 클릭 시 Store의 `insert()` 함수를 호출합니다.
    - 아래에서 생성할 Validator를 적용하여 유효성 검사를 수행합니다.

4.  **`update.vue` 생성 (`update()` 함수 존재 시)**

    - `[ModuleName]UpdateDto` 타입을 활용하여 수정 폼을 구성합니다.
    - `onMounted` 시 Store의 `detail()` 함수를 호출하여 초기 데이터를 가져옵니다.
    - '수정' 버튼 클릭 시 Store의 `update()` 함수를 호출합니다.
    - Validator를 적용하여 유효성 검사를 수행합니다.

5.  **`[모듈명].validator.ts` 생성 (`insert()` 또는 `update()` 함수 존재 시)**
    - `webapp/src/modules/test/pages/crud/_crud.validator.ts` 파일을 참고합니다.
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

**⚠️ Store에 해당 함수가 존재할 때만 생성되는 컴포넌트:**

- [ ] `webapp/src/modules/[모듈명]/pages/list.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/list-search.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/list-table.vue` (`paging()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/detail.vue` (`detail()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/insert.vue` (`insert()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/update.vue` (`update()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/pages/[모듈명].validator.ts` (`insert()` 또는 `update()` 함수 존재 시)
- [ ] `webapp/src/modules/[모듈명]/[모듈명].routes.ts` (생성된 컴포넌트 기준)

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

- **test/crud 폴더 기준**: 모든 구조와 디자인은 `webapp/src/modules/test/pages/crud/` 기준
- **Store 중심 개발**: UI는 Store의 상태와 함수를 중심으로 구현
- **조건부 컴포넌트 생성**: Store에 해당 함수가 존재할 때만 관련 컴포넌트 생성
- **SearchDto 완전 활용**: 검색 기능은 SearchDto의 모든 필드를 빠짐없이 활용
- **Type 기반 개발**: 모든 데이터 처리는 기존 Type 인터페이스 기반
- **완전한 연동**: 부분적 구현이 아닌 Store의 모든 기능을 완전 연동
