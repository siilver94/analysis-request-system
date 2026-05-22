# CLAUDE.md — 해석 의뢰 현황판 개발 컨텍스트

이 파일은 Claude AI가 프로젝트를 빠르게 파악하기 위한 전용 문서입니다.
새 대화창을 열 때 이 파일과 최신 YAML 파일을 함께 첨부하세요.
README.md는 사람용, 이 파일은 Claude 전용입니다.

---

## 프로젝트 개요

- 앱 이름: 연구소 해석 의뢰 현황판
- 회사: TYM (한국 농기계 회사) 중앙기술연구소
- 목적: 가상검증팀 해석 의뢰 요청 → 승인 → 수행 → 완료 전 과정 관리
- 플랫폼: Power Apps Canvas App → Microsoft Teams 탭 앱으로 배포
- 데이터: SharePoint List (주), Excel/OneDrive (보조)
- 라이선스: 무료 플랜 — Dataverse 불가, 프리미엄 커넥터 불가
- 개발자: 혼자 개발, 혼자 테스트

---

## 파일 구조

- PowerApps/WorkRequest_Screen.yaml : 앱 전체 코드 (화면 1개)
- sharepoint/sharepoint_schema.md : SharePoint 리스트 스키마 정의
- CLAUDE.md : 이 파일 (Claude 전용)
- CONTRIBUTING.md : 협업 규칙
- README.md : 프로젝트 소개 (사람용)

---

## 앱 구조 — 화면 1개

화면은 WorkRequest_Screen 하나에 모든 기능이 담겨 있음.

### 레이아웃

헤더 (Header_con_5)
  - 앱 제목 "연구소 해석 의뢰" + 우측 현재 사용자명

좌측 조회 필터 패널 (con_searchFilter) — X:8, W:271
  - 기간 드롭다운 (ddPeriodFilter) — colPeriodOptions 기반
  - 검색창 (txtSearch)
  - 완료 포함 토글 (tglIncludeCompleted)
  - 반려 포함 토글 (tglIncludeRejected)

칸반 3열 (Y:100, H:637)
  - 시작전 (Container7, X:293) — 주황 헤더 RGBA(217,119,6,1)
    - galStart — PendingTeamLeader, PendingVAApproval 상태
  - 진행중 (Container8, X:657) — 청록 헤더 RGBA(14,116,144,1)
    - galInProgress — InProgress 상태
  - 완료 (Container9, X:1014) — 초록 헤더 RGBA(34,139,34,1)
    - galCompleted — Completed, Rejected 상태 (토글로 제어)

신규등록 버튼 (new_btn, X:1234, Y:52)

팝업 레이어
  - 딤 배경 (OuterRactangle) — varShowRequestPopup=true일 때 표시
  - 팝업 본체 (conRequestPopup)
    - View 모드 (conRequestView) — 상세보기
      - 요약정보 (Container3)
      - 상세정보 (Container3_1)
      - 첨부파일 (Container3_2)
      - 역할별 액션 패널 (Y:510, H:100) — 4개 중 조건에 맞는 1개만 표시
        - conRequesterAction : 의뢰자 본인 + PendingTeamLeader
        - conLeaderAction : 팀장 + PendingTeamLeader
        - conVAApproveAction : VA승인자 + PendingVAApproval
        - conWorkerAction : InProgress 담당자 (⚠ 현재 Visible=true 하드코딩 버그)
      - 우측 패널 (conDetailLeft/Right) — 미완성 영역
    - New/Edit 모드 (confrmRequest)
      - frmAnalysisRequest : 의뢰 등록/수정 폼
      - txtRevisionComment : 수정 사유 (⚠ 팝업 컨테이너 바깥에 단독 배치됨)

---

## 역할 3가지

일반 의뢰자
  - 판단 변수: 없음 (기본)
  - 조건: 로그인한 모든 사용자
  - 가능한 작업: 의뢰 등록, 수정, 삭제 (PendingTeamLeader 단계만)

팀장
  - 판단 변수: varIsTeamLeader
  - 조건: varMyJobGrade = "팀장"
  - 가능한 작업: 팀장 승인(→PendingVAApproval), 반려

가상검증 승인자
  - 판단 변수: varIsVAApprover
  - 조건: 이메일 하드코딩 2명 (donggyu.lee@tym.world, E2603003@tym.world)
  - 가능한 작업: 담당자 지정 + VA승인(→InProgress), 반려

---

## Status 흐름

의뢰 등록 (SubmitForm → OnSuccess → Patch)
  → PendingTeamLeader (팀장 승인대기)
  → PendingVAApproval (가상검증 승인대기) — btnLeaderApprove
  → InProgress (진행중, AssigneeEmail 세팅) — btnVAApprove + drpAssignee
  → Completed (완료) — btnCompleteAnalysis (보고서 첨부 필요)
  또는 Rejected (반려) — 팀장 또는 VA승인자가 반려 처리

---

## 데이터소스 6개

Analysis_Request_DB — SharePoint List — 메인 의뢰 데이터
Analysis_Request_Revision_DB — SharePoint List — 수정 이력 (버전별 스냅샷)
Analysis_Request_Log_DB — SharePoint List — 처리 로그 (승인/반려/완료 등)
Analysis_Request_Files — Document Library — 의뢰 첨부파일
Analysis_Report_Files — Document Library — 최종 해석 보고서
UserInfo_DB — SharePoint List — 사용자 정보 (이름/팀/직급/이메일)

---

## 주요 전역변수

사용자 정보
  varMyEmail : Lower(User().Email)
  varMyName : UserInfo_DB.이름_한글 또는 User().FullName 파싱
  varMyDept : UserInfo_DB.부문
  varMyTeam : UserInfo_DB.팀
  varMyJobGrade : UserInfo_DB.직급2

역할
  varIsTeamLeader : Boolean
  varIsVAApprover : Boolean
  varVAApprover1Email : 하드코딩 donggyu.lee@tym.world
  varVAApprover2Email : 하드코딩 E2603003@tym.world

화면 상태
  varSelectedRequest : 현재 선택된 의뢰 레코드 (Record 타입)
  varRequestMode : "View" / "New" / "Edit"
  varShowRequestPopup : Boolean

필터
  varDateFrom / varDateTo : 기간 필터 날짜
  varPeriodFilter : "2026년" 또는 "전체"
  varIncludeCompleted : Boolean
  varIncludeRejected : Boolean

토스트 알림
  varShowToast : Boolean
  varToastMessage : String

기타
  varAppUrl : 앱 공유 URL (이메일 링크용)
  varSelectedRevision : 수정 이력에서 선택된 항목
  varShowDeleteConfirmPopup : Boolean

---

## 현재 알려진 버그 및 미완성 영역

버그 1 — conWorkerAction Visible 하드코딩
  위치: conWorkerAction > Visible 속성
  현재값: true (하드코딩)
  원래 의도: Status = "InProgress" && AssigneeEmail = varMyEmail 일 때만 표시
  증상: 담당자가 아닌 사람에게도 작업 처리 섹션이 보임

버그 2 — btnCompleteAnalysis DisplayMode 로직 불일치
  위치: btnCompleteAnalysis > DisplayMode 속성
  현재값: CountRows(varSelectedRequest.'첨부 파일') > 0 이면 활성화
  문제: 보고서는 frmReportUpload로 올리는데, 완료 버튼 조건이 일반 첨부파일 기준
  증상: 보고서 없이도 완료 버튼이 활성화될 수 있음

버그 3 — txtRevisionComment 위치 문제
  위치: WorkRequest_Screen 최상단 레벨에 단독 배치
  문제: 팝업 컨테이너(conRequestPopup) 바깥에 있어 Z-order 및 Visible 관리 주의 필요

미완성 4 — conDetailLeft / conDetailRight
  우측 패널 컨테이너 구조는 있으나 내부 콘텐츠 비어있음
  로그 이력, 진행 상태 표시 등 추가 예정

버그 5 — Stage_4 (새 수정 도착 Badge) Visible 조건 오류
  위치: galInProgress 내부 Stage_4 > Visible 속성
  문제: varSelectedRequest 기준으로 체크 중 → Gallery 내부에서는 ThisItem 기준이어야 함
  증상: 뱃지 표시 조건이 선택된 항목 기준으로 작동해 전체 갤러리 아이템에 잘못 적용됨

---

## Claude에게 — 개발 원칙

- 무료 플랜 제약 항상 준수: Dataverse 불가, 프리미엄 커넥터 불가
- 데이터소스: SharePoint List / Excel/OneDrive만 사용
- Power Automate: 표준 커넥터 기본 흐름만
- delegation 가능 함수 우선 (Filter, Search 등)
- delegation 불가 시 반드시 경고 후 대안 제시
- 수식 제공 시 적용 위치 반드시 명시 (컨트롤명 + 속성명 + 스크린명)
- 코드는 항상 실제 동작하는 완성본으로 제공
- 수식 아래에 "→ 이 수식이 하는 일: ..." 한 줄 요약 추가
- 성능 영향 수식은 최적화 방향 함께 제시
- 새 기능 요청 시 바로 코드 주지 말고 대안 먼저 제안 후 확정하면 구현
