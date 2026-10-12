# CLAUDE.md — 해석 의뢰 현황판 개발 컨텍스트

이 문서는 Claude와의 개발 세션 연속성을 위한 컨텍스트 파일입니다.
새 세션 시작 시 이 파일 + 최신 YAML(`WorkRequest_Screen.yaml`)을 함께 첨부하세요.

---

## 1. 프로젝트 기본 정보

- **앱**: TYM 중앙기술연구소 해석(CAE) 의뢰 현황판
- **플랫폼**: PowerApps Canvas App → Microsoft Teams 탭 배포
- **수식 언어**: Power Fx
- **데이터**: SharePoint List만 사용
- **GitHub**: https://github.com/siilver94/analysis-request-system

### 절대 제약 (무료 플랜)
- Dataverse 사용 불가 → SharePoint List만
- 프리미엄 커넥터·HTTP 요청·유료 기능 제안 금지
- 자동화는 표준 커넥터만 (Office365Outlook, OneDrive for Business)
- Word Online 커넥터의 PDF 변환은 프리미엄이라 사용 불가 → OneDrive Convert file(표준)로 대체

---

## 2. 데이터 소스 (SharePoint List)

| List | 역할 |
|--|--|
| Analysis_Request_DB | 의뢰 메인 (현재 상태) |
| Analysis_Request_Revision_DB | 수정 이력 + Revision별 첨부 |
| Analysis_Request_Log_DB | 처리 로그 |
| Analysis_Report_DB | 보고서 전용 (의뢰 첨부와 분리) |
| UserInfo_DB | 사용자 (이름_한글, e-mail, 부문, 팀, 직급2) |

### Analysis_Request_Log_DB 주요 컬럼
ActionByName, ActionByEmail, ActionRole, ActionAt, LogType, RequestIDKey, RequestNo, LogMessage, RelatedRevisionNo

**LogType Choice 값**: TeamLeaderApprove, (반려/배정 등 기존) + **AssigneeChanged**(담당자 변경), **TitleChanged**(제목 변경) 추가됨.
> Choice 컬럼에 없는 값을 Patch하면 에러 가능성 있으므로, 새 LogType 사용 시 SharePoint에서 선택지 먼저 추가할 것.

### Analysis_Request_Revision_DB 주요 컬럼
RequestIDKey, RequestNo, RevisionNo, RevisedByName, RevisedByEmail, RevisedAt,
AnalysisTitleSnapshot, ProductNameSnapshot, PartNoSnapshot, AnalysisReasonSnapshot,
AnalysisConditionSnapshot, DepartmentNameSnapshot, RequesterNameSnapshot, RevisionComment, 첨부 파일(Attachments)

> **중요**: 이 List의 커스텀 컬럼은 전부 SharePoint에서 Required 해제됨. 첨부 폼이 첨부만 제출하고 나머지는 OnSuccess에서 Patch하기 때문. Title(기본 필드)은 건드리지 않음.

---

## 3. 상태 흐름 & 역할

### Status
PendingTeamLeader → PendingVAApproval → InProgress → Completed (또는 Rejected)

### 역할
- 일반 의뢰자: `conRequesterAction` — 본인 + 미완료 단계 + `!varIsTeamLeader`
- 팀장/소장/이사: `conLeaderAction` — `varIsTeamLeader` = 직급2 in [팀장,소장,이사], PendingTeamLeader, 본인 의뢰도 결재 가능
- VA 승인자: `conVAApproveAction` — `varIsVAApprover` (이메일 하드코딩 2명: yoonju.park@tym.world, donghyeon.gim@tym.world), 담당자 지정 + 승인/반려
- 담당자: `conWorkerAction` — InProgress + AssigneeEmail = 내 이메일
- **가상검증팀 전원**: `varIsVATeamMember` (varMyTeam = "가상검증팀") — 전체 조회 + 제목 수정 + 완료 보고서 수정

### 조회 권한과 처리 권한 분리 [중요]
- **조회**: 가상검증팀 전원(`varIsVATeamMember`)이 3개 갤러리 전체 열람
- **처리(담당자 지정/승인/반려)**: VA 승인자 2명(`varIsVAApprover`)만
- 갤러리 Items의 최상위 조건이 `varIsAdmin || varIsVATeamMember`로 통합됨 (VA 승인자도 가상검증팀 소속이므로 별도 분기 제거)

---

## 4. 주요 변수

- 사용자: varMyEmail, varMyName, varMyDept, varMyTeam, varMyJobGrade, varMyInfo
- 역할: varIsTeamLeader, varIsVAApprover, varVAApprover1Email, varVAApprover2Email, **varIsAssigneeManager**(박윤주 전용, 담당자 변경 권한), **varIsVATeamMember**(가상검증팀 전원)
- 선택/모드: varSelectedRequest, varRequestMode(View/New/Edit), varSavedRequestMode, varShowRequestPopup, varSelectedRevision
- 신규 등록: varNewRequestNo, varNewRequestIDKey, varRequesterLeaderInfo
- Revision 첨부 분리: varLatestRevision, varNewRevisionNo, varSnapshotTitle, varSnapshotProduct, varSnapshotPart, varSnapshotReason, varSnapshotCondition, varRevisionComment
- 담당자 변경: **varOldAssigneeEmail, varOldAssigneeName, varNewAssigneeEmail, varNewAssigneeName** (Refresh 전 선캡처용)
- 제목 인라인 수정: **varTitleEditMode, varTitleEditInput**
- 보고서: varReportRecord, varReportAttachCount, **varReportEditMode**(완료 보고서 편집 토글)
- PDF: **varPdfResult**(흐름 반환값, .pdflink 필드), varApproveLog(팀장 승인 로그 조회)
- 필터: varDateFrom, varDateTo, varPeriodFilter, varIncludeCompleted, varIncludeRejected
- 기타: varShowToast, varToastMessage

---

## 5. 이메일 발송 패턴 [중요]

표준 커넥터 Office365Outlook. 무료 플랜 OK.

    Office365Outlook.SendEmailV2(
        받는사람,
        제목,
        "<div style='font-family:Segoe UI;'> ... HTML 본문 ... </div>",
        { IsHtml: true, Importance: "High" }   // 또는 "Normal"
    )

발송 위치: 등록(팀장 승인요청) / 팀장 승인·반려 / VA 승인(담당자+의뢰자)·반려 / 진행 저장(의뢰자) / 작업 완료(의뢰자 + 가상검증팀 전원) / 담당자 변경(신규 담당자 + 의뢰자).

- **앱 바로가기 링크는 본문에 넣지 않음** (보안 정책, 게시 시 URL이 매번 바뀜, App.LaunchUrl 미지원).
- 가상검증팀 전원 발송은 `ForAll(Filter(UserInfo_DB, 팀 = "가상검증팀"), SendEmailV2(...))` 패턴. 팀 5명 소규모라 속도 무관, 담당자 본인 포함 발송 OK.

---

## 6. 의뢰번호 체계 (RequestNo)

형식: `CAE` + 연도2자리 + 순번3자리 (예: CAE26039)

frmAnalysisRequest.OnSuccess 신규 블록에서 생성:

    Set(
        varNewRequestNo,
        "CAE" &
        Text(Year(Now()) - 2000, "00") &
        Text(CountRows(Filter(Analysis_Request_DB, Year(SubmittedAt) = Year(Now()))) + 38, "000")
    )

- `+38` 오프셋으로 올해 CAE26039부터 시작 (이전엔 +32 → 33번)
- 내년 자동 CAE27001 시작
- 동시 등록 시 중복 가능성 있으나 소규모라 무시

---

## 7. 해석 의뢰서 PDF 출력 (Power Automate 방식) [핵심]

### 배경
Canvas App의 `Print()`는 화면 뷰포트 크기(기본 768)에 종속되어 긴 문서 하단이 잘림. HtmlViewer(=HTML 텍스트 컨트롤)의 AutoHeight를 켜도, 앱 전체를 감싸는 최상위 프레임이 고정 크기라 Teams/브라우저/Ctrl+P 모두 실패. → Canvas 앱 구조적 한계로 확정, Power Automate로 분리.

### 흐름: `해석보고서_PDF생성`
1. Trigger: When Power Apps calls a flow (V2) — 텍스트 입력 11개
   (text=RequestNo, text_1=RequesterName, text_2=RequesterTeam, text_3=SubmittedAtText,
    text_4=ExpectedFinishDateText, text_5=ProductName, text_6=PartNo, text_7=AnalysisReason,
    text_8=AnalysisCondition, text_9=ApproverName, text_10=ApprovedDateText)
2. OneDrive Create file: /해석보고서_임시/report.html (HTML은 @{triggerBody()?['text_N']}로 값 삽입)
3. OneDrive Convert file: Target type = PDF (File 파라미터엔 Path가 아니라 **Id/Identifier** 동적콘텐츠)
4. OneDrive Create file: {RequestNo}.pdf 저장 (File Content = Convert file의 File Content)
5. OneDrive Create share link: File = Create file 1의 **Id**, Link type=View, **scope=Organization**
6. Respond to a PowerApp: 출력 PdfLink = Create share link의 Web URL

### 앱 호출 (btnPrintReport.OnSelect)
- varApproveLog = LookUp(Log_DB, ... LogType="TeamLeaderApprove") 먼저 조회
- '해석보고서_PDF생성'.Run(11개 인자, 날짜는 Text()로 문자열 변환, Coalesce로 blank 처리)
- Launch(varPdfResult.pdflink)  ← 반환 필드명은 소문자 pdflink

### 트러블슈팅 기록 (반복 방지)
- Convert file / Create share link의 File 파라미터는 **Path(경로 문자열) 아니라 Id/Identifier**. Path 넣으면 "파일 ID 유효하지 않음" 400.
- 공유 scope Anonymous → 테넌트 정책상 차단("링크 통한 공유 미사용"). **Organization으로 해결**.
- 흐름이 My Flows에만 있으면 Teams 기반 Studio '흐름 추가'에 안 뜸 → **솔루션 생성 후 기존 항목 추가**로 포함해야 인식됨.
- 앱과 흐름이 **같은 환경**에 있어야 함 (환경 ID로 확인, 이름만 보고 판단 금지).

### HTML 템플릿 (압축판, 한 페이지)
- body margin 24px 32px, 기본 font 10px, 라벨 컬럼 width 70px, 값 셀 font 11px
- 테이블 셀 padding 6px 8px, 섹션 margin-bottom 16px, line-height 1.4
- 결재란 width 300px, 푸터 테이블 border-top 제거(이중선 해소)
- 동적값은 Code view에서 @{triggerBody()?['text_N']} 형태로 통째 교체 가능

---

## 8. 이번 세션 신규 구현 기능

### 8-1. 진행 중 담당자 변경 (박윤주 팀장 전용)
- `varIsAssigneeManager` = varMyEmail = yoonju.park@tym.world
- `conVAApproveAction` Visible 확장: (PendingVAApproval && varIsVAApprover) || (InProgress && varIsAssigneeManager)
- InProgress 시: btnVAApprove/btnLeaderReject_1/txtVARejectReason Visible = PendingVAApproval만, drpAssignee DefaultSelectedItems = 현 담당자
- `btnChangeAssignee`(신규): **변수 선캡처 패턴 필수** — drpAssignee.Selected는 Refresh 후 초기화되므로 Patch/메일 전에 varNewAssigneeEmail/Name에 미리 Set (안 하면 메일 미발송 버그)
- Patch + 로그(AssigneeChanged) + 신규 담당자·의뢰자 메일

### 8-2. 제목 인라인 수정 (가상검증팀 전원)
- Container2에 iconEditTitle(연필, varIsVATeamMember && !varTitleEditMode), txtTitleEdit, btnTitleSave, btnTitleCancel
- Text1_1(제목) Visible = !varTitleEditMode, Width 850
- btnTitleSave: Title Patch + 로그(TitleChanged) + Refresh + 토스트
- 모든 상태(진행중/완료 포함)에서 가능

### 8-3. 가상검증팀 전체 조회
- galStart/galInProgress/galCompleted Items 최상위 조건을 `varIsAdmin || varIsVATeamMember`로 통합 (기존 varIsVAApprover 분기 흡수)
- galCompleted는 관련자 조건에 varIsVATeamMember 한 줄 추가
- 처리 버튼 권한은 varIsVAApprover 유지 (조회/처리 분리)

### 8-4. 상세내역 의뢰번호 표시
- Container2에 lblRequestNoDetail 추가, X = Stage_3.X + Stage_3.Width + 12, Y = 42 (상태 배지 오른쪽)

### 8-5. 완료 시 가상검증팀 전원 알림
- btnCompleteAnalysis.OnSelect에 ForAll(Filter(UserInfo_DB, 팀="가상검증팀"), SendEmailV2 팀 전체용 문구) 추가
- 의뢰자 메일도 상세 HTML로 개선

### 8-6. 완료 건 보고서 수정 (가상검증팀) — B안 채택
- frmReportView(FormViewer, 읽기전용) Visible = !varReportEditMode 로 유지
- frmReportEdit(신규 **Form**, DefaultMode=Edit, Item=varReportRecord, Visible=varReportEditMode) + 내부 첨부 파일_ReportEditCard(ClassicAttachmentsEdit) + DataCardValueReportEdit
- frmReportEdit.OnSuccess: Refresh + varReportRecord 재조회 + FinalReportAttached 동기화 + varReportEditMode=false + 토스트
- 버튼: btnReportModify(수정, Set true), btnReportSaveEdit(SubmitForm(frmReportEdit)), btnReportCancelEdit(ResetForm+Refresh+false)
- **배경**: FormViewer는 DefaultMode 없음(조회 전용). DataCardValue3_1.Attachments 직접 Patch는 "Table (Attachment) 형식 불일치" 에러 → Form 컨트롤 + SubmitForm 경유만 안전. 알림 메일 없음.

---

## 9. Revision별 첨부파일 분리 (이전 세션 구현, 유지)

### 모델
"Revision 행 = 그 시점 저장된 버전(새 상태 + 그때 올린 파일)"

    최초 등록  → Rev.1 생성 (LastRevisionNo=1)
    1차 수정   → Rev.2 생성 (새 텍스트 + 새 첨부, "새로 올리기")
    현재/최신  = 가장 높은 RevisionNo

### 핵심 기술 사실
- **저장된 SharePoint 첨부는 List 간 복사 불가** (Value가 URL 참조). "새로 올리기" 채택.
- 첨부는 반드시 Form을 통해 저장 (Patch로 직접 첨부 불가).

### 저장 흐름 (폼 체이닝)
    btnRequestSave.OnSelect
      → 검증 + 스냅샷 변수 저장 + varNewRevisionNo 계산 + SubmitForm(frmAnalysisRequest)
    frmAnalysisRequest.OnSuccess
      → New/Edit DB 업데이트 + 로그 + 이메일 + SubmitForm(frmRevisionAttach)
    frmRevisionAttach.OnSuccess
      → Revision 행 메타 Patch + Refresh + 폼리셋 + 팝업닫기 + 토스트

### 관련 컨트롤
- `frmRevisionAttach`: Revision DB 바인딩, New 모드, 새 첨부 업로드용
- `frmRequestAttachmentView`: View 모드 첨부 조회 → DataSource=Analysis_Request_Revision_DB, Item=varLatestRevision
- `frmPrevRevAttachView`: Edit 모드 이전 버전 첨부 읽기 전용(클릭 다운로드)
- `첨부 파일_DataCard3`(frmAnalysisRequest 내): Visible=false
- Edit 팝업 좌우 분할: 좌300 이전버전, 우517 새 업로드 / New 모드는 전체폭 821

### varLatestRevision 세팅 위치
galStart>Rectangle1, galInProgress>Rectangle1_3, galCompleted>Rectangle1_5 의 OnSelect 맨 앞:

    Set(varLatestRevision, LookUp(Analysis_Request_Revision_DB,
        RequestIDKey = ThisItem.RequestIDKey && RevisionNo = ThisItem.LastRevisionNo))

---

## 10. 알려진 패턴 & 주의사항

- **Form 첨부 저장**: ThisItem + Parent.Default 조합 필수. Items = Parent.Default.
- **FormViewer는 편집 불가**: 편집 필요 시 Form 컨트롤 별도 사용 + SubmitForm.
- **컨트롤 Selected 값 선캡처**: drpAssignee.Selected 등은 Refresh 후 초기화 → Patch/메일 전에 변수로 미리 Set (담당자 변경 메일 버그 재발 방지).
- **보고서(Analysis_Report_DB)**: 의뢰 첨부와 완전 분리. frmReportUpload는 SubmitForm만, OnSuccess에서 후처리.
- **타이밍**: SubmitForm → OnSuccess 순서. Refresh 후 LookUp 해야 최신값. 후처리는 OnSuccess에 몰아넣어 타이밍 꼬임 방지.
- **Power Automate 흐름 File 파라미터**: Path 아니라 Id/Identifier. 공유 scope는 Organization. 흐름은 솔루션 포함 + 앱과 동일 환경.
- **galRevisionHistory에서 Rev별 첨부 조회**: 보류 상태. 추후 별도 화면 예정.

---

## 11. 작업 컨벤션

- 수식 앞에 `=` 붙이지 않기 (복붙용)
- 코드 줄 때 적용 위치 명시 (컨트롤 + 속성 + 스크린)
- 수식 아래 "→ 이 수식이 하는 일" 한 줄 요약
- 템플릿 주석·인라인 가이드 텍스트는 코드에 포함하지 않음 (로직 버그 유발 전례)
- 메일 발신자 인사말: "안녕하십니까, 김은성 주임 입니다"로 시작
- 진행 방식: 문제 진단 → 현재 상태 확인 → 단계별 수정 → 스크린샷 확인, 한 번에 하나씩
- 변경사항은 GitHub로 버전 관리, *.msapp는 .gitignore

---

## 12. 최근 변경 이력 (이번 세션)

### 오류 수정
- **보고서 출력 하단 짤림**: Print() → Power Automate + OneDrive HTML→PDF 변환 방식으로 근본 해결
- **RequestNo 시작값**: +32 → +38 (CAE26039부터)
- **메일 앱 바로가기 링크 제거**: 게시 시 URL 변동 + 보안 정책, App.LaunchUrl 미지원

### 기능 개발
- 진행 중 담당자 변경 (박윤주 팀장 전용, 변수 선캡처 패턴)
- 제목 인라인 수정 (가상검증팀 전원, 연필 토글)
- 가상검증팀 전체 진행 내역 조회 (조회/처리 권한 분리)
- 상세내역 의뢰번호 표시 (lblRequestNoDetail)
- 완료 시 가상검증팀 전원 알림 (ForAll)
- 완료 건 보고서 수정 (frmReportEdit, B안)

### 구조 변경
- **PrintScreen 삭제** (Power Automate PDF 방식으로 전환)
- Analysis_Request_Log_DB LogType에 AssigneeChanged, TitleChanged 추가

### 보류 / 다음 작업 후보
- galRevisionHistory Rev 클릭 시 첨부 조회 화면 (별도 화면 구성 예정)
- View_Screen 안정성 추가 검증
