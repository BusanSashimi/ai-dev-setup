# 프론트엔드 Store 자동 생성 가이드

## 🎯 목적

백엔드 API를 기반으로 Vue3 Store (상태관리)를 자동 생성합니다.

- API 호출 함수와 상태 변수 포함
- 프로젝트 표준 규칙을 완벽히 준수하는 Store 생성

## 🔧 사용 방법

**명령어:**

```bash
/run [모듈명]
```

**사용 예시:**

```bash
/run member-auth
/run itax-client
/run board-notice
```

**📝 모듈명 자동 변환:**

언더스코어(`_`)를 포함한 모듈명은 자동으로 케밥 케이스(`-`)로 변환됩니다:

```bash
/run member_auth         # → member-auth 모듈로 처리
/run itax_client         # → itax-client 모듈로 처리
/run board_notice_mgmt   # → board-notice-mgmt 모듈로 처리
```

## 📋 프로젝트 핵심 규칙

### 작업 환경

**작업 폴더**: `webapp/`
**백엔드 API 폴더**: `api/`

### 파일 명명 규칙

- 스토어: `[모듈명].store.ts`
- 타입: `[모듈명].type.ts`
- 테스트: `[모듈명].test.ts`
- 파일명은 **케밥 케이스(kebab-case)** 사용

### Store 개발 규칙

- **Option API 방식**으로 코딩 (store 파일 전용)
- 모든 REST API 호출은 **store 클래스**를 통해 수행
- `@/modules/common/service/api.services.ts`의 `useApi` 사용
- 조회 데이터(list, detail)는 상태 변수로 관리
- 등록, 수정, 삭제는 파라미터로 전달받아 처리

### Store 함수명 규칙

- **페이징 목록 조회**: `paging()` (메인 목록 조회, 페이징 포함)
- **키워드 검색**: `list(keyword)` (키워드 기반 목록 조회)
- **상세 조회**: `detail()`
- **등록**: `insert()`
- **수정**: `update()`
- **사용여부 변경**: `updateUse()`
- **삭제**: `softDelete()`
- **상세 데이터 초기화**: `detailDataInit()`

### 타입 정의 규칙

- 주석은 필드 왼쪽에 배치
- 감사 칼럼은 마지막에 배치
- 인터페이스: `[모듈명]`, `[모듈명]SearchDto`, `[모듈명]PagingDto`, `[모듈명]InsertDto`, `[모듈명]UpdateDto`

### 감사 칼럼 처리

- **isUse**: `updateUse()` 함수로 처리
- **isDelete**: `softDelete()` 함수로 처리

## 🤖 AI 수행 작업 (사용자가 "실행" 입력 시)

1. **모듈명 정규화**

   - 언더스코어(`_`)를 하이픈(`-`)으로 자동 변환
   - `member_auth` → `member-auth`로 정규화

2. **Controller 파일 분석**

   - **백엔드 API 폴더**에서 `[모듈명].controller.ts` 파일 검색
   - **실제 구현된 API 엔드포인트만** 파악 (미구현 API 제외)
   - API 요청/응답 형식 분석, 관련 Type 파일 확인

3. **모듈 폴더 구조 생성**

   - `webapp/src/modules/[모듈명]/` 디렉토리 생성 (기존 모듈 폴더에 파일 생성 금지)

4. **기존 Store 파일 확인**

   - 해당 모듈에 이미 Store 파일이 있는지 확인
   - 기존 Store가 있으면 누락된 기능 파악, 없으면 새로 생성

5. **타입 인터페이스 생성/업데이트**

   - `webapp/src/modules/[모듈명]/type/[모듈명].type.ts` 생성
   - `api/src/modules/[모듈명]/type/[모듈명].type.ts` 파일을 기준으로 백엔드 타입과 100% 일치하게 생성
   - 기본 타입, SearchDto, PagingDto, InsertDto, UpdateDto 생성

6. **Store 파일 생성/완성**

   - `webapp/src/modules/[모듈명]/run/[모듈명].store.ts` 생성
   - `webapp/src/modules/test-data/run/test-data.store.ts` 파일 형식 참고
   - Option API 방식 사용

7. **상태 변수 정의**

   - `webapp/src/modules/test-data/run/test-data.store.ts` 파일 형식 참고

8. **API 호출 함수 구현**

   - `useApi` 사용하여 HTTP 요청 처리
   - **Controller에서 실제 구현된 API만 Store에 포함**
   - 미구현 API 제외 (빈 함수, 주석 처리, TODO 등)

9. **테스트 코드 생성**

   - `webapp/src/modules/[모듈명]/test/[모듈명].test.ts` 생성
   - `webapp/src/modules/test-data/test/test-data.test.ts` 파일 형식 참고
   - CRUD API 호출 테스트 포함

10. **백엔드 서버 구동 및 테스트 실행**

    - 백앤드 작업폴더로 이동 후 `cd api && npm start` 실행
    - 10초 대기 후 작업 폴더로 이동 `npx vitest run [테스트파일]` 실행하여 검증

11. **품질 검증 및 최종 확인**
    - **Store 파일 품질 검증**: 타입 정의, API 함수 구현 완성도 확인
    - **린트 검사**: `npm run lint:fix` 실행
    - **타입 체크**: `npx vue-tsc --noEmit` 실행
    - **최종 테스트**: 백엔드 서버 구동 상태에서 테스트 재실행

## ✅ 완료 확인

### 생성된 파일 체크리스트

- [ ] `webapp/src/modules/[모듈명]/type/[모듈명].type.ts`
- [ ] `webapp/src/modules/[모듈명]/run/[모듈명].store.ts`
- [ ] `webapp/src/modules/[모듈명]/test/[모듈명].test.ts`

### Store 기능 체크리스트

**Controller에서 확인된 API를 기준으로 해당하는 기능만 체크:**

- [ ] 페이징 목록 조회 (`paging()`) 함수 구현
- [ ] 키워드 검색 (`list(keyword)`) 함수 구현
- [ ] 상세 조회 (`detail()`) 함수 구현
- [ ] 등록 (`insert()`) 함수 구현
- [ ] 수정 (`update()`) 함수 구현
- [ ] 사용여부 변경 (`updateUse()`) 함수 구현
- [ ] 삭제 (`softDelete()`) 함수 구현
- [ ] 상세 데이터 초기화 (`detailDataInit()`) 함수 구현

### 품질 검증 단계

1. **파일 생성 완료 확인**

   ```bash
   ls -la /src/modules/[모듈명]/run/
   ls -la /src/modules/[모듈명]/type/
   ```

2. **타입 체크 및 린트 검사**

   ```bash

   npx vue-tsc --noEmit
   npm run lint:fix
   ```

3. **최종 테스트 검증**

   1. 백엔드 api 서버 구동(별도 터미널에서 실행)

   ```bash
   cd api
   npm start
   ```

   2. Frontend 테스트 실행 (메인 터미널에서 잠시 대기후 실행)

   ```bash
   cd webapp
   npx vitest run /src/modules/[모듈명]/test/[모듈명].test.ts
   ```

### 최종 품질 체크리스트

- [ ] Store 파일이 정상 생성됨
- [ ] 타입 파일이 정상 생성됨
- [ ] 백엔드 타입과 100% 일치
- [ ] 타입 구조 설계 원칙 준수 (SearchDto 상속 구조)
- [ ] 모든 API 함수 구현 완료
- [ ] API 함수별 정확한 타입 적용
- [ ] Pinia Store 패턴 준수 (Option API)
- [ ] 타입 에러 없음 (`vue-tsc` 통과)
- [ ] 린트 에러 없음 (`npm run lint:fix` 통과)
- [ ] 테스트 100% 통과
- [ ] API 호출 함수가 실제 백엔드와 연결 가능
