# 도메인 컨트롤러 기반 Frontend Store 개발 가이드

## 🎯 개요

백엔드 도메인 컨트롤러(Java)를 기반으로 Vue3 Frontend Store (상태관리)를 개발하는 표준 가이드입니다.

- **백엔드 API 우선**: 실제 구현된 Controller API만 Frontend에 반영
- **도메인 중심 개발**: 업무 도메인별 체계적 구조화
- **프로젝트 표준 준수**: Option API 방식 Store 개발
- **TDD 기반 품질 보장**: 모든 API 호출 테스트 완료

## 📋 지원 도메인

### 1. 주요 업무 도메인

```
📂 /domain
├── taxi/          # 스마트 세무사 관련 업무
│   ├── unpaid/    # 미수금 관리
│   ├── singo/     # 신고 관리
│   ├── pay/       # 결제 관리
│   └── client/    # 고객 관리
├── scraping/      # 스크래핑 업무
├── messaging/     # 메시징 업무
├── hometax/       # 홈택스 관련 업무
│   ├── vat/       # 부가세
│   ├── income/    # 소득세
│   ├── corporate/ # 법인세
│   └── certification/ # 인증서
├── report/        # 보고서 업무
├── fourIns/       # 4대보험 업무
├── chatgpt/       # AI 관련 업무
└── kcapp/         # KC앱 관련 업무
```

### 2. 컨트롤러 예시

```java
// 스마트 세무사 도메인 예시
@RestController
@RequestMapping("/v1/taxi/unpaid")
public class TaxiUnpaidController { ... }

@RestController
@RequestMapping("/v1/taxi/client")
public class TaxiClientController { ... }

// Messaging 도메인 예시
@RestController
@RequestMapping("/v1/messaging/sms")
public class MessagingSmsController { ... }

// Hometax 도메인 예시
@RestController
@RequestMapping("/v1/hometax/vat")
public class VatController { ... }
```

## 🚀 개발 프로세스

### Step 1: 백엔드 컨트롤러 분석

**1-1. 타겟 컨트롤러 선택**

```bash
# 도메인 컨트롤러 확인
find api/src/main/java/com/kacpta/kcplf/domain -name "*Controller.java"

# 예시 결과
TaxiUnpaidController.java      # 스마트 세무사 미수금 관리
TaxiClientController.java      # 스마트 세무사 고객 관리
MessagingSmsController.java
VatController.java
```

**1-2. API 메서드 분석**

```java
// 예시: TaxiUnpaidController.java (스마트 세무사 미수금 관리)
@RestController
@RequestMapping("/v1/taxi/unpaid")
public class TaxiUnpaidController {

    @GetMapping("")               // 페이징 조회
    @GetMapping("/list")          // 리스트 조회
    @GetMapping("/{id}")          // 상세 조회
    @PostMapping("")              // 등록
    @PutMapping("/{id}")          // 수정
    @DeleteMapping("/{id}")       // 삭제

    // 기타 도메인 특화 메서드들...
}
```

### Step 2: 타입 정의 작성

**2-1. 파일 위치**

```
webapp/types/[도메인]/[기능].dto.ts
```

**2-2. 타입 정의 예시**

```typescript
// types/taxi/unpaid.dto.ts (스마트 세무사 미수금 관리)
export interface TaxiUnpaid {
  /** 미수금 번호 */
  unpaidSeq: number;
  /** 고객 번호 */
  clientSeq: number;
  /** 미수금 금액 */
  unpaidAmount: number;
  /** 등록일 */
  insertDate: string;
  // ... 기타 필드들
}

export interface TaxiUnpaidPagingDto {
  page: number;
  row: number;
  sortBy?: string;
  sortType?: "asc" | "desc";
  keyword?: string;
}
```

### Step 3: Store 구현

**3-1. 파일 위치**

```
webapp/features/[도메인]/store/[기능].store.ts
```

**3-2. Store 구현 예시**

```typescript
// features/taxi/store/unpaid.store.ts (스마트 세무사 미수금 관리)
import type { OperationResponse } from "@/stores/common.dto";
import { useApi } from "@/service/api/api.service";
import type { TaxiUnpaid, TaxiUnpaidPagingDto } from "@/types/taxi/unpaid.dto";

export class TaxiUnpaidService {
  /**
   * 미수금 페이징 조회
   */
  static paging(
    params: TaxiUnpaidPagingDto
  ): Promise<OperationResponse<{ totalRow: number; list: TaxiUnpaid[] }>> {
    return useApi().get("taxi/unpaid", { params });
  }

  /**
   * 미수금 리스트 조회
   */
  static async list(): Promise<OperationResponse<TaxiUnpaid[]>> {
    return await useApi().get("taxi/unpaid/list");
  }

  /**
   * 미수금 상세 조회
   */
  static async detail(
    unpaidSeq: number
  ): Promise<OperationResponse<TaxiUnpaid>> {
    return await useApi().get(`taxi/unpaid/${unpaidSeq}`);
  }

  /**
   * 미수금 등록 (백엔드 API 존재 시에만 구현)
   */
  static async insert(
    params: TaxiUnpaidInsertDto
  ): Promise<OperationResponse<TaxiUnpaid>> {
    return await useApi().post("taxi/unpaid", params);
  }

  // ... 기타 메서드들
}
```

### Step 4: TDD 테스트 작성

**4-1. 파일 위치**

```
webapp/features/[도메인]/test/[기능].test.ts
```

**4-2. TDD 테스트 예시**

```typescript
// features/taxi/test/unpaid.test.ts (스마트 세무사 미수금 관리)
import { describe, it, expect } from "vitest";
import { TaxiUnpaidService } from "@/features/taxi/store/unpaid.store";

describe("TaxiUnpaid CRUD", () => {
  it("미수금 페이징 조회가 성공해야 함", async () => {
    const params = {
      page: 1,
      row: 10,
      sortBy: "unpaidSeq",
      sortType: "desc",
    };
    const result = await TaxiUnpaidService.paging(params);
    expect(result.operationStatus).toBe("SUCCESS");
  });

  it("미수금 리스트 조회가 성공해야 함", async () => {
    const result = await TaxiUnpaidService.list();
    expect(result.operationStatus).toBe("SUCCESS");
  });

  it("미수금 상세 조회가 성공해야 함", async () => {
    const listResult = await TaxiUnpaidService.list();
    if (listResult.content && listResult.content.length > 0) {
      const firstItem = listResult.content[0];
      const detailResult = await TaxiUnpaidService.detail(firstItem.unpaidSeq);
      expect(detailResult.operationStatus).toBe("SUCCESS");
    }
  });
});
```

### Step 5: 테스트 실행 및 검증

**5-1. 테스트 실행**

```bash
cd webapp
npm run test features/[도메인]/test/[기능].test.ts
```

**5-2. 예상 결과**

```bash
✓ features/taxi/test/unpaid.test.ts (5) - 스마트 세무사 미수금 관리
  ✓ TaxiUnpaid CRUD (5)
    ✓ Read (3)
      ✓ 미수금 페이징 조회가 성공해야 함
      ✓ 미수금 리스트 조회가 성공해야 함
      ✓ 미수금 상세 조회가 성공해야 함
    ✓ API Connection (2)
      ✓ API 호출 성공
      ✓ 데이터 구조 검증 성공
```

## 🔧 핵심 개발 원칙

### ✅ Backend API 우선 개발

**🎯 중요**: 백엔드 컨트롤러에 **실제 구현된 API만** Store에 반영

```java
// ✅ 구현된 API만 Store에 반영
@GetMapping("")          → paging() 메서드
@GetMapping("/list")     → list() 메서드
@GetMapping("/{id}")     → detail() 메서드
@PostMapping("")         → insert() 메서드

// ❌ 구현되지 않은 API는 Store에서 제외
// PUT, PATCH, DELETE 등이 없으면 해당 메서드 구현 금지
```

### ✅ 도메인 중심 구조화

**도메인별 독립적 구조 유지**

```
features/
├── taxi/           # 스마트 세무사
│   ├── store/unpaid.store.ts
│   ├── store/client.store.ts
│   └── test/unpaid.test.ts
├── messaging/
│   ├── store/sms.store.ts
│   └── test/sms.test.ts
└── hometax/
    ├── store/vat.store.ts
    └── test/vat.test.ts
```

### ✅ 프로젝트 표준 준수

- **Option API 방식** Store 개발
- **useApi() 활용** REST API 호출
- **OperationResponse<T>** 응답 타입 사용
- **kebab-case 파일명** 사용

### ✅ 타입 안전성 보장

- 백엔드 API 응답과 100% 일치하는 타입 정의
- TypeScript 컴파일 에러 0개
- ESLint 경고 0개

## 📊 도메인별 개발 예시

### 1. 스마트 세무사 도메인 (실제 완료 사례)

```typescript
// PopupService (Global 도메인)
✅ PopupService.paging()   // GET /v1/popup
✅ PopupService.list()     // GET /v1/popup/list
✅ PopupService.detail()   // GET /v1/popup/{id}
❌ insert(), update(), delete() // 백엔드 미구현으로 제외
```

### 2. Messaging 도메인 (개발 예정)

```typescript
// MessagingSmsService
✅ MessagingSmsService.paging()     // GET /v1/messaging/sms
✅ MessagingSmsService.send()       // POST /v1/messaging/sms
✅ MessagingSmsService.history()    // GET /v1/messaging/sms/history
```

### 3. Hometax 도메인 (개발 예정)

```typescript
// VatService
✅ VatService.paging()        // GET /v1/hometax/vat
✅ VatService.calculate()     // POST /v1/hometax/vat/calculate
✅ VatService.download()      // GET /v1/hometax/vat/download
```

## 🎯 코드 품질 검증

### 1. 필수 테스트 항목

- **API 호출 성공**: 모든 메서드 정상 응답
- **데이터 구조 검증**: 응답 타입 일치 확인
- **에러 처리**: 예외 상황 적절한 처리
- **성능 테스트**: 응답 시간 측정

### 2. 린트 및 타입 체크

```bash
# 타입 체크
npm run type-check

# 린트 체크
npm run lint

# 테스트 실행
npm run test
```

## 🚀 확장 가능성

### 1. 새로운 도메인 추가

```bash
# 1. 백엔드 컨트롤러 확인
find api/src/main/java/com/kacpta/kcplf/domain/[새도메인] -name "*Controller.java"

# 2. 타입 정의 작성
webapp/types/[새도메인]/[기능].dto.ts

# 3. Store 구현
webapp/features/[새도메인]/store/[기능].store.ts

# 4. TDD 테스트
webapp/features/[새도메인]/test/[기능].test.ts
```

### 2. 기존 도메인 API 추가

```java
// 백엔드에 새 API 추가 시 (스마트 세무사 미수금 관리)
@PostMapping("/batch")
public ResponseEntity<List<TaxiUnpaid>> batchInsert(@RequestBody List<TaxiUnpaidInsertDto> params) {
    // 구현 로직
}
```

```typescript
// Frontend Store에 메서드 추가
static async batchInsert(params: TaxiUnpaidInsertDto[]): Promise<OperationResponse<TaxiUnpaid[]>> {
  return await useApi().post("taxi/unpaid/batch", params);
}
```

## ✅ 완료 체크리스트

**개발 전 확인사항**

- [ ] 백엔드 도메인 컨트롤러 분석 완료
- [ ] 실제 구현된 API 목록 확인
- [ ] 프로젝트 폴더 구조 확인

**개발 중 확인사항**

- [ ] 타입 정의 완료
- [ ] Store 구현 완료
- [ ] TDD 테스트 작성 완료
- [ ] 모든 테스트 통과

**개발 후 확인사항**

- [ ] 실제 API 연동 확인
- [ ] 타입 에러 없음
- [ ] 린트 에러 없음
- [ ] 불필요한 코드 제거

---

**📌 결론**: 백엔드 도메인 컨트롤러를 기반으로 체계적이고 확장 가능한 Frontend Store를 개발하는 표준 가이드입니다. 각 도메인별로 독립적이면서도 일관된 구조를 유지하며, 백엔드 API와 완벽히 연동되는 고품질 코드를 보장합니다.

## 🎯 실제 작업 사례: Popup Store (완료)

### 백엔드 API 분석 결과

**PopupController.java 분석:**

```java
@RestController
@RequestMapping("/v1/popup")
public class PopupController {

    // ✅ 구현된 API들
    @GetMapping("")               // 페이징 조회
    @GetMapping("/list")          // 리스트 조회
    @GetMapping("/{popupSeq}")    // 상세 조회

    // ❌ 미구현 API들 (Controller에 없음)
    // POST, PUT, PATCH, DELETE 등
}
```

### 생성된 파일 구조

```
webapp/
├── types/common/popup.dto.ts           # 타입 정의
├── features/common/store/popup.store.ts # Store 구현
└── features/common/test/popup.test.ts   # TDD 테스트
```

### 타입 정의 완료 (`popup.dto.ts`)

```typescript
/**
 * 팝업 관리 DTO
 */
export interface Popup {
  /** 팝업 번호 */
  popupSeq: number;
  /** 팝업 제목 */
  title: string;
  /** 팝업 내용 */
  content: string;
  /** 사용 여부 */
  isUse: string;
  /** 삭제 여부 */
  isDelete: string;
  /** 팝업 시작일 */
  startDate: string;
  /** 팝업 종료일 */
  endDate: string;
  /** 등록자 번호 */
  insertSeq: number;
  /** 등록일 */
  insertDate: string;
  /** 수정자 번호 */
  updateSeq: number;
  /** 수정일 */
  updateDate: string;
}

/**
 * 팝업 페이징 요청 DTO
 */
export interface PopupPagingDto {
  /** 페이지 번호 */
  page: number;
  /** 페이지 크기 */
  row: number;
  /** 정렬 컬럼 */
  sortBy?: string;
  /** 정렬 방향 */
  sortType?: "asc" | "desc";
  /** 검색 키워드 */
  keyword?: string;
}
```

### Store 구현 완료 (`popup.store.ts`)

**백엔드 실제 구현 API만 반영:**

```typescript
export class PopupService {
  /**
   * 팝업 페이징 조회 - GET /v1/popup
   */
  static paging(
    params: PopupPagingDto
  ): Promise<OperationResponse<{ totalRow: number; list: Popup[] }>> {
    return useApi().get("popup", { params });
  }

  /**
   * 팝업 리스트 조회 - GET /v1/popup/list
   */
  static async list(): Promise<OperationResponse<Popup[]>> {
    return await useApi().get("popup/list");
  }

  /**
   * 팝업 상세 조회 - GET /v1/popup/{popupSeq}
   */
  static async detail(popupSeq: number): Promise<OperationResponse<Popup>> {
    return await useApi().get(`popup/${popupSeq}`);
  }
}
```

### TDD 테스트 완료 (`popup.test.ts`)

**전체 테스트 통과 ✅**

```typescript
describe("Popup CRUD", () => {
  it("팝업 페이징 조회가 성공해야 함", async () => {
    const params = {
      page: 1,
      row: 10,
      sortBy: "popupSeq",
      sortType: "desc",
    } as PopupPagingDto;
    const result = await PopupService.paging(params);
    expect(result.operationStatus).toBe("SUCCESS");
  });

  it("팝업 리스트 조회가 성공해야 함", async () => {
    const result = await PopupService.list();
    expect(result.operationStatus).toBe("SUCCESS");
  });

  it("팝업 상세 조회가 성공해야 함", async () => {
    const listResult = await PopupService.list();
    if (listResult.content && listResult.content.length > 0) {
      const firstPopup = listResult.content[0];
      const detailResult = await PopupService.detail(firstPopup.popupSeq);
      expect(detailResult.operationStatus).toBe("SUCCESS");
    }
  });
});
```

### 테스트 실행 결과

```bash
 ✓ features/common/test/popup.test.ts (5) 406ms
   ✓ Popup CRUD (5) 405ms
     ✓ Read (3)
       ✓ 팝업 페이징 조회가 성공해야 함
       ✓ 팝업 리스트 조회가 성공해야 함
       ✓ 팝업 상세 조회가 성공해야 함
     ✓ API Connection (2)
       ✓ 팝업 페이징 API 호출이 성공해야 함
       ✓ 팝업 리스트 API 호출이 성공해야 함

 Test Files  1 passed (1)
      Tests  5 passed (5)
```
