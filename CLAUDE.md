# CLAUDE.md — 해석 의뢰 현황판 개발 컨텍스트

이 파일은 Claude AI가 프로젝트를 빠르게 파악하기 위한 전용 문서입니다.
새 대화창을 열 때 이 파일과 최신 YAML 파일을 함께 첨부하세요.

---

## 프로젝트 개요

- 앱 이름: 연구소 해석 의뢰 현황판
- 회사: TYM (한국 농기계 회사) 중앙기술연구소
- 목적: 가상검증팀 해석 의뢰 요청 → 승인 → 수행 → 완료 전 과정 관리
- 플랫폼: Power Apps Canvas App → Microsoft Teams 탭 앱으로 배포
- 데이터: SharePoint List만 사용
- 라이선스: 무료 플랜 — Dataverse 불가, 프리미엄 커넥터 불가
- 개발자: 혼자 개발, 혼자 테스트

---

## 앱 구조 — 화면 2개

### 1. WorkRequest_Screen (메인 화면)
칸반 3열 + 좌측 필터 패널 + 팝업 레이어. 모든 핵심 기능이 여기 담겨 있음.

### 2. PrintScreen (인쇄 화면)
완료된 의뢰건의 의뢰서를 인쇄/PDF 저장하기 위한 화면.
- HtmlText 컨트롤(htPrintReport)로 의뢰서 렌더링
- OnVisible에서 Print() 자동 실행
- 돌아가기 버튼(btnBack)은 인쇄 중 숨김 처리 (varIsPrinting 변수로 제어)
- 인쇄 내용: 요약정보 + 상세정보만 포함 (첨부파일, 처리이력 등은 제외)

---

## WorkRequest_Screen 레이아웃

### 헤더 (Header_con_5)
- 앱 제목 "연구소 해석 의뢰" + 우측 현재 사용자명

### 좌측 조회 필터 패널 (con_searchFilter) — X:8, W:271
- 기간 드롭다운 (ddPeriodFilter) — colPeriodOptions 기반
- 검색창 (txtSearch)
- 완료 포함 토글 (tglIncludeCompleted) — OnVisible에서 기본값 true 설정
- 반려 포함 토글 (tglIncludeRejected) — OnVisible에서 기본값 true 설정

### 칸반 3열 (Y:100, H:637)
- 시작전 (Container7, X:293) — 주황 헤더 RGBA(217,119,6,1)
  - galStart — PendingTeamLeader, PendingVAApproval 상태
- 진행중 (Container8, X:657) — 청록 헤더 RGBA(14,116,144,1)
  - galInProgress — InProgress 상태
- 완료 (Container9, X:1014) — 초록 헤더 RGBA(34,139,34,1)
  - galCompleted — Completed, Rejected 상태 (토글로 제어)

### 신규등록 버튼 (new_btn, X:1234, Y:52)

### 팝업 레이어
- 딤 배경 (OuterRactangle) — varShowRequestPopup=true일 때 표시
- 팝업 본체 (conRequestPopup)
  - View 모드: Width 1360, Height 760
  - New/Edit 모드: Width 930, Height 715

---

## conRequestView (View 모드) 상세 구조

### 좌측 영역
- conDetailLeft (요약정보) — 요청자/팀/요청일
- Container3_1 (상세정보) — 품명/품번/해석사유/해석조건
- conRejectReason (반려사유) — Status="Rejected"일 때만 표시
- Container3_2 (첨부파일) — frmRequestAttachmentView 사용
- 역할별 액션 패널 (Y:564, 4개 중 조건에 맞는 1개만 표시)

### 우측 패널 (conDetailRight) — X:1008, W:330
- conStatusTimeline (처리 현황) — 4단계 타임라인
  - 등록 / 팀장 승인 / VA 승인 / 완료(또는 반려)
  - 각 단계 ● 아이콘 색상: 완료=초록, 미완료=회색, 반려=빨강
  - 날짜는 Analysis_Request_Log_DB에서 LogType별 LookUp으로 조회
- conAssigneeInfo (진행 정보) — 담당자/진행률/예상완료일/작업메모
- conReportSection (해석 보고서) — Status="Completed"일 때만 표시
  - frmReportView FormViewer로 Analysis_Report_DB 첨부파일 조회
- 수정 이력 (galRevisionHistory + conRevDetail)
  - 갤러리 클릭 시 conRevDetail에 상세 표시 (varSelectedRevision 기반)
  - TemplateSize 고정 44px, 갤러리 위에 lblRevHistoryTitle 별도 배치

---

## 역할별 액션 패널 4개

### conRequesterAction (일반 의뢰자)
- Visible: 의뢰자 본인 && Status in [PendingTeamLeader, PendingVAApproval, InProgress] && !varIsTeamLeader
- btnEditRequest (수정) : 완료/반려 전까지 항상 표시
- btnDeleteRequest (삭제) : PendingTeamLeader일 때만 표시 (회수)

### conLeaderAction (팀장/소장/이사)
- Visible: varIsTeamLeader && Status="PendingTeamLeader" && (RequesterLeaderEmail=내 이메일 || RequesterEmail=내 이메일)
- btnLeaderApprove (승인) → Status=PendingVAApproval, 메일 발송
- btnLeaderReject (반려) + txtLeaderRejectReason → Status=Rejected
- btnEditRequest_Leader (수정) : 본인 의뢰일 때만 표시
- btnDeleteRequest_Leader (삭제) : 본인 의뢰 + PendingTeamLeader일 때만

### conVAApproveAction (VA 승인자)
- Visible: varIsVAApprover && Status="PendingVAApproval"
- drpAssignee (담당자 드롭다운) — UserInfo_DB에서 팀="가상검증팀" 필터
- btnVAApprove (승인) — 담당자 선택 시에만 활성화, Status=InProgress, AssigneeEmail 세팅
- btnLeaderReject_1 (반려) + txtVARejectReason

### conWorkerAction (담당자)
- Visible: Status="InProgress" && AssigneeEmail=내 이메일
- Height: 150 (제약)
- 구조: 진행률 슬라이더 + 예상완료일 + 작업메모 + 보고서 업로드 영역
- sldProgressPct (진행률)
- dpExpectedFinishDate (예상완료일)
- txtWorkerMemo (작업메모) — Default: Coalesce(varSelectedRequest.WorkerMemo, "")
- btnSaveWorkProgress (저장)
- btnCompleteAnalysis (작업 완료)
  - DisplayMode: FinalReportAttached=true일 때만 활성화
  - 활성화 시 초록색 강조
- 보고서 업로드 영역 (별도 우측에 배치)

---

## 보고서 업로드 (Analysis_Report_DB 분리)

### 배경
초기에는 Analysis_Request_DB의 기본 첨부파일을 의뢰 첨부와 보고서가 공유하는 구조였음.
이 경우 의뢰자 첨부와 작업자 보고서가 섞이는 문제 발생 → 별도 List로 분리.

### 구조
- frmReportUpload Form 사용
  - DataSource: Analysis_Report_DB
  - Item: varReportRecord
  - DefaultMode: If(IsBlank(varReportRecord), FormMode.New, FormMode.Edit)
- DataCardValue1.Items: Parent.Default (ThisItem 기반 자동 갱신 핵심)
- 첨부 파일_DataCard1.Default: ThisItem.'첨부 파일'
- btnReportAdd 클릭 시:
  - SubmitForm(frmReportUpload)
  - OnSuccess에서 메타데이터 Patch + FinalReportAttached 업데이트

### 핵심 학습 사항 (반복 발생 문제)
- Form의 Item이 바뀌어도 변수(varReportRecord)로 Items를 직접 지정하면 갱신 안 됨
- 반드시 ThisItem과 Parent.Default 조합 사용해야 자동 갱신됨
- frmAnalysisRequest의 첨부파일 처리 방식과 동일하게 맞춰야 안정적

---

## Status 흐름

의뢰 등록 (SubmitForm → OnSuccess → Patch)

↓

PendingTeamLeader (팀장 승인대기)

↓ btnLeaderApprove

PendingVAApproval (가상검증 승인대기)

↓ btnVAApprove + drpAssignee

InProgress (진행중, AssigneeEmail 세팅)

↓ btnCompleteAnalysis (FinalReportAttached=true 필요)

Completed (완료)

또는 어느 단계든 Rejected (반려) — 팀장 또는 VA승인자가 반려 처리


---

## 데이터소스 (실제 사용 4개)

- Analysis_Request_DB — 메인 의뢰 데이터
- Analysis_Request_Revision_DB — 수정 이력 (버전별 스냅샷)
- Analysis_Request_Log_DB — 처리 로그
- Analysis_Report_DB — 해석 보고서 (작업자 업로드 전용, 별도 분리)
- UserInfo_DB — 사용자 정보 (이름/팀/직급/이메일)

미사용 (앱에 연결만 되어있음):
- Analysis_Request_Files (Document Library)
- Analysis_Report_Files (Document Library)

---

## Analysis_Request_DB 주요 컬럼 (추가/변경분)

- RequesterTeam : Single line text — 등록 시점 팀명 스냅샷 (신규 추가)
- FinalReportAttached : Yes/No — 보고서 첨부 여부 (Analysis_Report_DB와 연동)

---

## Analysis_Report_DB 컬럼

- Title : Single line text — "{RequestNo}_보고서"
- RequestIDKey : Single line text — 메인 의뢰 연결 키
- RequestNo : Single line text — 의뢰번호
- UploadedByName : Single line text
- UploadedByEmail : Single line text
- UploadedAt : Date and Time
- 첨부파일 : SharePoint List 기본 제공

---

## 주요 전역변수

### 사용자 정보
- varMyEmail : Lower(User().Email)
- varMyName : UserInfo_DB.이름_한글 또는 User().FullName 파싱
- varMyDept : UserInfo_DB.부문
- varMyTeam : UserInfo_DB.팀
- varMyJobGrade : UserInfo_DB.직급2

### 역할
- varIsTeamLeader : varMyJobGrade in ["팀장", "소장", "이사"]
- varIsVAApprover : 이메일 하드코딩 2명 체크
- varVAApprover1Email : donggyu.lee@tym.world
- varVAApprover2Email : E2603003@tym.world

### 화면 상태
- varSelectedRequest : 현재 선택된 의뢰 레코드
- varRequestMode : "View" / "New" / "Edit"
- varShowRequestPopup : Boolean
- varSelectedRevision : 수정 이력 갤러리에서 선택된 항목
- varReportRecord : 현재 의뢰의 보고서 레코드 (Analysis_Report_DB LookUp 결과)
- varReportAttachCount : 보고서 첨부파일 수 (FinalReportAttached 판단용)
- varIsPrinting : PrintScreen 인쇄 중 여부 (돌아가기 버튼 숨김용)

### 필터
- varDateFrom / varDateTo : 기간 필터 날짜
- varPeriodFilter : "2026년" 또는 "전체"
- varIncludeCompleted : Boolean (OnVisible에서 true 초기화)
- varIncludeRejected : Boolean (OnVisible에서 true 초기화)

### 토스트 알림
- varShowToast : Boolean
- varToastMessage : String

### 기타
- varAppUrl : 앱 공유 URL (이메일 링크용)
- varShowDeleteConfirmPopup : Boolean

---

## 갤러리 클릭 시 OnSelect 공통 패턴

galStart, galInProgress, galCompleted 내부 Rectangle OnSelect:

```
Set(varReportRecord, Blank());
Refresh(Analysis_Report_DB);
Set(varSelectedRequest, ThisItem);
Set(varRequestMode, "View");
Set(varShowRequestPopup, true);
Set(
varReportRecord,
LookUp(
Analysis_Report_DB,
RequestIDKey = ThisItem.RequestIDKey
)
);
If(
IsBlank(varReportRecord),
Patch(
Analysis_Request_DB,
ThisItem,
{ FinalReportAttached: false }
)
);
```
→ varReportRecord를 먼저 Blank 처리해야 Form 캐시 문제 안 생김

---

## 인쇄 기능

- 버튼: btnPrintReport (상세보기 팝업 헤더 우측 상단)
- Visible: varSelectedRequest.Status.Value = "Completed"
- OnSelect: Navigate(PrintScreen)
- PrintScreen 구성:
  - htPrintReport (HtmlText) — 의뢰서 전체 렌더링
  - btnBack — 돌아가기, Visible=!varIsPrinting
- OnVisible: Set(varIsPrinting, true); Print(); Set(varIsPrinting, false)
- HTML 스타일: 인라인 스타일만 사용 (class 기반 스타일은 HtmlText에서 불안정)
- 내용: 헤더(타이틀 + 의뢰번호) + 요약정보 테이블 + 상세정보 테이블 + 푸터
- 라벨/값 구분: border-right 1px (가로선 덮지 않도록 1px 유지)

---

## 현재 알려진 미완성/개선 예정 사항

1. VA승인자 관리 — 현재 이메일 하드코딩
   → 추후 UserInfo_DB에 IsVAApprover 컬럼 추가해서 동적 관리 예정

2. Analysis_Request_Files, Analysis_Report_Files Document Library 미사용
   → 활용 여부 추후 검토

---

## Claude에게 — 개발 원칙

- 무료 플랜 제약 항상 준수: Dataverse 불가, 프리미엄 커넥터 불가
- 데이터소스: SharePoint List만 사용
- Power Automate: 표준 커넥터 기본 흐름만
- delegation 가능 함수 우선 (Filter, Search 등)
- delegation 불가 시 반드시 경고 후 대안 제시
- 수식 제공 시 적용 위치 반드시 명시 (컨트롤명 + 속성명 + 스크린명)
- **수식 앞에 '=' 붙이지 말 것** (복사 후 바로 붙여넣기 가능하도록)
- 코드는 항상 실제 동작하는 완성본으로 제공
- 수식 아래에 "→ 이 수식이 하는 일: ..." 한 줄 요약 추가
- 새 기능 요청 시 바로 코드 주지 말고 대안 먼저 제안 후 확정하면 구현
- Form 내부 첨부파일 처리는 반드시 ThisItem + Parent.Default 조합 사용

  

