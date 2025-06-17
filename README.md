# AI 개발 자동화 도구 가이드

프로젝트를 위한 자동화된 백엔드/프론트엔드 CRUD 코드 생성 시스템입니다.

> 🤖 **AI 모델 요구사항**: **Claude 4 Sonnet Thinking** 모델을 사용하세요.

## 📁 프로젝트 구조

```
root/  #root (예시로 backoffice 프로젝트면 backoffice가 되고 itax이면 itax가 됨.)
├── api/              # 백엔드 (Node.js + TypeScript)
│   └── src/
│       └── modules/             # 기능별 모듈
├── webapp/           # 프론트엔드 (Vue3 + Nuxt UI)
└── ai-dev/                      # 🤖 AI 개발 자동화 도구
    ├── README.md                # 이 파일
    ├── backend-crud.md          # 백엔드 CRUD 자동 생성 가이드
    ├── frontend-store.md        # 프론트엔드 Store 자동 생성 가이드
    └── frontend-ui.md           # 프론트엔드 UI 자동 생성 가이드
```

## 🚀 사용 방법

### 📝 기본 명령어

```bash
/run [테이블명]

해당 파일을 전체를 읽고 가이드를 숙지하여 진행
```

> **모든 단계에서 동일한 명령어를 사용합니다. 순차적으로 실행하면 됩니다.**

---

### 1단계: 백엔드 CRUD 생성

**목적**: 데이터베이스 테이블을 기반으로 백엔드 CRUD API를 완전 자동 생성

- Controller, Service, DAO, Type, Validator, TDD 테스트까지 포함
- 프로젝트 표준 규칙을 완벽히 준수

### 2단계: 프론트엔드 Store 생성

**목적**: 백엔드 API를 기반으로 Vue3 Store (상태관리)를 자동 생성

- API 호출 함수와 상태 변수 포함
- 백엔드 타입과 100% 일치하는 타입 생성

### 3단계: 프론트엔드 UI 생성

**목적**: 기존 Store와 Type을 기반으로 완전한 CRUD UI 컴포넌트를 자동 생성

- 실제 동작하는 완전한 CRUD 기능 구현
- NuxtUI v3, TailwindCSS v4 적용
- 모바일 & 데스크톱 반응형 디자인
