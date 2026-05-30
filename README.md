# 해석 의뢰 현황판 (Analysis Request System)

TYM 중앙기술연구소 가상검증팀의 **해석 의뢰 관리 시스템**입니다.
의뢰 등록 → 팀장 승인 → 가상검증팀 승인 → 작업 수행 → 보고서 업로드 → 완료까지 전 과정을 관리합니다.

---

## 기술 스택

- **Frontend**: Power Apps Canvas App
- **Backend**: SharePoint List (5개)
- **배포**: Microsoft Teams 탭 앱
- **라이선스**: Microsoft 365 무료 플랜 (Dataverse / 프리미엄 커넥터 미사용)

---

## 주요 기능

### 1. 4단계 워크플로우

    PendingTeamLeader → PendingVAApproval → InProgress → Completed
                                                 ↓ (어느 단계에서든)
                                             Rejected

### 2. 4가지 역할

- **일반 의뢰자**: 의뢰 등록·수정·삭제 (PendingTeamLeader 단계만)
- **팀장/소장/이사**: 팀장 승인·반려, 본인 의뢰 시 수정 가능
- **VA 승인자**: 담당자 지정 + 승인·반려
- **담당자**: 진행률 입력, 보고서 업로드, 작업 완료

### 3. 수정 이력 관리

- Analysis_Request_Revision_DB에 버전별 스냅샷 저장
- 갤러리에서 클릭하여 과거 버전 상세 조회

### 4. 칸반 뷰

- 시작전 / 진행중 / 완료 3열 구조
- 기간·검색·완료포함·반려포함 필터

### 5. 인쇄 기능

- 완료된 의뢰서 PDF 출력용 별도 화면 (PrintScreen)
- HtmlText 인라인 스타일 기반 렌더링

---

## 프로젝트 구조

    analysis-request-system/
    ├── PowerApps/
    │   └── WorkRequest_Screen.yaml      # 앱 전체 코드 (메인 + 인쇄 화면)
    ├── sharepoint/
    │   └── sharepoint_schema.md         # SharePoint 리스트 스키마
    ├── docs/                            # (예정) 스크린샷, 사용 매뉴얼
    ├── CLAUDE.md                        # AI 협업용 프로젝트 컨텍스트
    ├── CONTRIBUTING.md                  # 협업 규칙
    └── README.md

---

## 데이터소스 (SharePoint List)

| 리스트명 | 용도 |
|---------|------|
| Analysis_Request_DB | 메인 의뢰 데이터 |
| Analysis_Request_Revision_DB | 수정 이력 (버전 스냅샷) |
| Analysis_Request_Log_DB | 상태 변경 로그 |
| Analysis_Report_DB | 해석 보고서 (작업자 업로드) |
| UserInfo_DB | 사용자 정보 (이름·팀·직급·이메일) |

---

## 개발 규칙

- SharePoint List만 사용 (Dataverse, 프리미엄 커넥터 금지)
- Delegation 가능 함수 우선 사용
- Form 첨부파일 처리는 반드시 `ThisItem` + `Parent.Default` 조합 사용
- 자세한 컨텍스트는 [CLAUDE.md](./CLAUDE.md) 참조

---

## 라이선스

내부 연구소 용도. 외부 공개 불가.
