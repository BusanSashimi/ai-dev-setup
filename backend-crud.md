# 백엔드 CRUD 자동 생성 가이드

## 🎯 목적

데이터베이스 테이블을 기반으로 완전한 백엔드 CRUD API를 자동 생성합니다.

- Controller, Service, DAO, Type, Validator, TDD 테스트까지 포함
- 프로젝트 표준 규칙을 완벽히 준수하는 코드 생성

## 🔧 사용 방법

**명령어:**

```bash
/run [테이블명]              # 기본값: file=N (파일 업로드 없음)
/run [테이블명] file=Y       # 파일 업로드 포함
```

**사용 예시:**

```bash
/run member_auth             # 파일 업로드 없는 회원 권한 테이블 (기본값)
/run board_notice file=Y     # 파일 업로드 있는 게시판 테이블
/run itax_client             # 파일 업로드 없는 세무 클라이언트 테이블 (기본값)
```

**파라미터 설명:**

- `[테이블명]`: 데이터베이스 테이블명 (스네이크 케이스)
- `file=Y`: 파일 업로드 기능 포함 (생략 시 file=N 기본값)

## 📋 프로젝트 핵심 규칙

### 파일 명명 규칙

- 컨트롤러: `[모듈명].controller.ts`
- 서비스: `[모듈명].service.ts`
- DAO: `[모듈명].dao.ts`
- 타입: `[모듈명].type.ts`
- 테스트: `[모듈명].test.ts`
- Validator: `[모듈명].validator.ts`
- 파일명은 **케밥 케이스(kebab-case)** 사용

### 함수명 규칙

- **Controller**: getAll, getOne, insert, update, updateUse, softDelete, hardDelete
- **Service**: findPaging, findList, findOne, detailAll, detailOne, insert, update, updateUse, softDelete, hardDelete
- **DAO**: findPaging, findList, findOne, insert, update, updateUse, softDelete, hardDelete

### 데이터베이스 규칙

- Boolean 값: `CHAR(1)` 사용, 'Y'/'N' 값
- 금액: `DECIMAL(14,0)` (최대 100조)
- 날짜: `DATETIME` 타입
- 순번: `seq` (id 대신 사용)
- 감사 칼럼: is_use, is_delete, insert_seq, insert_date, update_seq, update_date

### 타입 정의 규칙

- 옵셔널 타입(`?`) 사용하지 않기
- `null`, `undefined` 타입 허용하지 않기
- 주석은 필드 왼쪽에 배치
- 감사 칼럼은 마지막에 배치
- 인터페이스: `[테이블명]`, `[테이블명]PagingDto`, `[테이블명]InsertDto`, `[테이블명]UpdateDto`

### 에러 처리

- 기능적 오류: HTTP 200 + JSON 응답
- 시스템 예외: `throw new ErrorHandler()` 사용
- try-catch에서 타입 가드 사용

## 🤖 AI 수행 작업 (사용자가 "실행" 입력 시)

1. **스키마 확인**

   - `cd api && npm run check-schema [테이블명]` 실행
   - 테이블 구조 및 컬럼 정보 분석
   - 컬럼 주석(COMMENT)이 '삭제'인 필드 제외

2. **모듈 폴더 생성**

   - 테이블명 전체를 케밥 케이스로 변환하여 독립 모듈 생성
   - `api/src/modules/[테이블명-케밥케이스]/` 폴더 구조 생성

3. **타입 인터페이스 생성 (작업폴더안에서 꼭 작업!! 중요!)**

   - `api/src/modules/[테이블명-케밥케이스]/type/[테이블명-케밥케이스].type.ts`로 파일 생성
   - `api/src/modules/test/type/test-data.type.ts` 파일 형식 참고
   - 4개 인터페이스 생성: 기본, PagingDto, InsertDto, UpdateDto

4. **DAO 생성 (작업폴더안에서 꼭 작업!! 중요!)**

   - `api/src/modules/[테이블명-케밥케이스]/dao/[테이블명-케밥케이스].dao.ts`로 파일 생성
   - `api/src/modules/test/dao/test-data.dao.ts` 파일 형식 참고
   - `sql-template-strings` 라이브러리 사용
   - 감사 칼럼은 쿼리 마지막에 배치

5. **Service 생성 (작업폴더안에서 꼭 작업!! 중요!)**

   - `api/src/modules/[테이블명-케밥케이스]/service/[테이블명-케밥케이스].service.ts`로 파일 생성
   - `api/src/modules/test/service/test-data.service.ts` 파일 형식 참고
   - static 메서드 사용
   - 파일 업로드 여부에 따라 함수 조정

6. **Validator 생성 (작업폴더안에서 꼭 작업!! 중요!)**

   - `api/src/modules/[테이블명-케밥케이스]/controller/[테이블명-케밥케이스].validator.ts`로 파일 생성
   - `class-validator` 데코레이터 사용
   - 데이터베이스 제약사항 반영

7. **Controller 생성 (작업폴더안에서 꼭 작업!! 중요!)**

   - `api/src/modules/[테이블명-케밥케이스]/controller/[테이블명-케밥케이스].controller.ts`로 파일 생성
   - `api/src/modules/test/controller/test-data.controller.ts` 파일 형식 참고
   - `routing-controllers` 라이브러리 사용
   - RESTful API 설계

8. **TDD 테스트 생성 (작업폴더안에서 꼭 작업!! 중요!)**

   - `api/src/modules/[테이블명-케밥케이스]/test/[테이블명-케밥케이스].test.ts`로 파일 생성
   - `api/src/modules/test/test/test-data.test.ts` 파일 형식 참고
   - CRUD 기본 테스트 포함

9. **테스트 실행**

   - `npx jest [테스트파일]` 실행하여 검증
   - 모든 테스트 통과 확인

10. **품질 검증 및 최종 확인**

    - **파일 품질 검증**: 생성된 모든 파일의 코드 품질 재점검
      - 타입 정의 완성도 확인
      - 함수 구현 완성도 확인
      - 주석 및 네이밍 규칙 준수 확인
    - **린트 검사 및 수정**: `npm run lint:fixed` 실행하여 코드 스타일 통일
    - **빌드 검사 및 수정**: `npm run build` 실행하여 빌드가 잘되는지 체크
    - **최종 테스트 재실행**: `npx jest [테스트파일]` 한번 더 실행하여 최종 검증
    - **완성도 확인**: 모든 CRUD 기능이 완벽히 동작하는지 최종 점검

## 🔧 파라미터 옵션

### 파일 업로드 처리

**file=N (파일 업로드 없음):**

- **Service**: findOne 함수만 적용, detailAll/detailOne/fileSetting 함수 제거
- **Validator**: fileList, imageList 제거
- **Controller**: getOne 함수에 service의 findOne 함수 적용

**file=Y (파일 업로드 포함):**

- **Service**: detailAll/detailOne/fileSetting 함수 포함, #parentCode/#parentCodeImage 함수 적용
- **Validator**: fileList, imageList 적용
- **Controller**: service의 detailOne 함수 호출

## ✅ 완료 확인

1. **린트 검사 및 자동 수정**

   ```bash
   npm run lint:fixed
   ```

2. **최종 테스트 검증**
   ```bash
   npx jest api/src/modules/[모듈명]/test/[모듈명].test.ts
   ```

### 최종 품질 체크리스트

- [ ] 타입 정의가 완벽함 (옵셔널 타입 없음, 주석 완비)
- [ ] 함수 구현이 완성됨 (모든 CRUD 기능)
- [ ] 네이밍 규칙 준수함 (케밥 케이스, 표준 함수명)
- [ ] 린트 에러 없음 (`npm run lint:fixed` 통과)
- [ ] 테스트 100% 통과 (2회 검증 완료)
- [ ] 코드 스타일 일관성 확보
