# CLAUDE.md — 해석 의뢰 현황판 개발 컨텍스트

이 문서는 Claude와의 개발 세션 연속성을 위한 컨텍스트 파일입니다.
새 세션 시작 시 이 파일 + 최신 YAML(`WorkRequest_Screen.yaml`, `PrintScreen.yaml`)을 함께 첨부하세요.

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
- 자동화는 표준 커넥터만 (Office365Outlook 등)
- Power Automate는 기본 흐름만

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

### Analysis_Request_Revision_DB 주요 컬럼
RequestIDKey, RequestNo, RevisionNo, RevisedByName, RevisedByEmail, RevisedAt,
AnalysisTitleSnapshot, ProductNameSnapshot, PartNoSnapshot, AnalysisReasonSnapshot,
AnalysisConditionSnapshot, DepartmentNameSnapshot, RequesterNameSnapshot, RevisionComment, 첨부 파일(Attachments)

> **중요**: 이 List의 커스텀 컬럼은 전부 SharePoint에서 Required 해제됨. 첨부 폼이 첨부만 제출하고 나머지는 OnSuccess에서 Patch하기 때문. Title(기본 필드)은 건드리지 않음.

---

## 3. 상태 흐름 & 역할

### Status
PendingTeamLeader → PendingVAApproval → InProgress → Completed (또는 Rejected)

### 역할 (4종)
- 일반 의뢰자: `conRequesterAction` — 본인 + 미완료 단계 + `!varIsTeamLeader`
- 팀장/소장/이사: `conLeaderAction` — `varIsTeamLeader` = 직급2 in [팀장,소장,이사], PendingTeamLeader, 본인 의뢰도 결재 가능
- VA 승인자: `conVAApproveAction` — `varIsVAApprover` (이메일 하드코딩 2명: yoonju.park@tym.world, donghyeon.gim@tym.world), 담당자 지정 + 승인/반려
- 담당자: `conWorkerAction` — InProgress + AssigneeEmail = 내 이메일

---

## 4. 주요 변수

- 사용자: varMyEmail, varMyName, varMyDept, varMyTeam, varMyJobGrade, varMyInfo
- 역할: varIsTeamLeader, varIsVAApprover, varVAApprover1Email, varVAApprover2Email
- 선택/모드: varSelectedRequest, varRequestMode(View/New/Edit), varSavedRequestMode, varShowRequestPopup, varSelectedRevision
- 신규 등록: varNewRequestNo, varNewRequestIDKey, varRequesterLeaderInfo
- Revision 첨부 분리(신규): varLatestRevision, varNewRevisionNo, varSnapshotTitle, varSnapshotProduct, varSnapshotPart, varSnapshotReason, varSnapshotCondition, varRevisionComment
- 보고서: varReportRecord, varReportAttachCount
- 필터: varDateFrom, varDateTo, varPeriodFilter, varIncludeCompleted, varIncludeRejected
- 기타: varAppUrl, varShowToast, varToastMessage

---

## 5. 이메일 발송 패턴 [중요]

표준 커넥터 Office365Outlook. 무료 플랜 OK.

    Office365Outlook.SendEmailV2(
        받는사람,
        제목,
        "<div style='font-family:Segoe UI;'> ... HTML 본문 ... </div>",
        { IsHtml: true, Importance: "High" }   // 또는 "Normal"
    )

발송 위치: 등록(팀장 승인요청) / 팀장 승인·반려 / VA 승인(담당자+의뢰자) · 반려 / 진행 저장(의뢰자) / 작업 완료(의뢰자).
본문에는 항상 `varAppUrl` 앱 바로가기 링크 포함.

---

## 6. 의뢰번호 체계 (RequestNo)

형식: `CAE` + 연도2자리 + 순번3자리 (예: CAE26033)

frmAnalysisRequest.OnSuccess 신규 블록에서 생성:

    Set(
        varNewRequestNo,
        "CAE" &
        Text(Year(Now()) - 2000, "00") &
        Text(CountRows(Filter(Analysis_Request_DB, Year(SubmittedAt) = Year(Now()))) + 32, "000")
    )

- 올해 기존 32건 반영 → 33번부터 시작
- 내년 자동 CAE27001 시작
- 동시 등록 시 중복 가능성 있으나 소규모라 무시

---

## 7. Revision별 첨부파일 분리 [이번 세션 핵심 구현]

### 모델
"Revision 행 = 그 시점 저장된 버전(새 상태 + 그때 올린 파일)"

    최초 등록  → Rev.1 생성 (원본 텍스트 + 첨부 직접 업로드)
    1차 수정   → Rev.2 생성 (새 텍스트 + 새 첨부, "새로 올리기")
    현재/최신  = 가장 높은 RevisionNo

### 핵심 기술 사실
- **저장된 SharePoint 첨부는 List 간 복사 불가** (Value가 URL 참조). → "이어받기" 불가, "새로 올리기" 채택
- 업로드 순간의 파일은 메모리에 실체가 있어 Form으로 저장 가능
- 첨부는 반드시 Form을 통해 저장 (Patch로 직접 첨부 불가)

### 저장 흐름 (폼 체이닝)

    btnRequestSave.OnSelect
      → 검증 + 스냅샷 변수 저장(varSnapshot*, varNewRevisionNo) + SubmitForm(frmAnalysisRequest)
    frmAnalysisRequest.OnSuccess
      → New/Edit DB 업데이트 + 로그 + 이메일 + SubmitForm(frmRevisionAttach)
    frmRevisionAttach.OnSuccess
      → Revision 행에 메타데이터 Patch(Self.LastSubmit) + Refresh + 폼리셋 + 팝업닫기 + 토스트

### 관련 컨트롤
- `frmRevisionAttach` : Revision DB 바인딩, New 모드, 새 첨부 업로드용 (confrmRequest 내)
- `frmRequestAttachmentView` : View 모드 첨부 조회 → DataSource를 **Analysis_Request_Revision_DB**, Item = `varLatestRevision`
- `frmPrevRevAttachView` : Edit 모드에서 이전 버전 첨부 읽기 전용 표시 (클릭 시 다운로드 가능)
- `첨부 파일_DataCard3` (frmAnalysisRequest 내) : Visible = false 처리 (더 이상 메인 DB에 첨부 저장 안 함)

### Edit 팝업 레이아웃 (좌우 분할)
- 좌측 300: 이전 버전 첨부 (frmPrevRevAttachView, 읽기 전용)
- 우측 517: 새 첨부 업로드 (frmRevisionAttach)
- New 모드: frmRevisionAttach 전체 너비(821), 이전 버전 영역 숨김
- 안내 문구: "※ 이전 버전 파일을 유지하려면 새로 업로드해 주세요." (Edit 모드, 빨간색)

### varLatestRevision 세팅 위치
galStart > Rectangle1, galInProgress > Rectangle1_3, galCompleted > Rectangle1_5 의 OnSelect 맨 앞에서:

    Set(varLatestRevision, LookUp(Analysis_Request_Revision_DB,
        RequestIDKey = ThisItem.RequestIDKey && RevisionNo = ThisItem.LastRevisionNo))

### 신규 등록 시 LastRevisionNo
- 최초 등록 시 LastRevisionNo = 1 (Rev.1 생성됨). 과거엔 0이었음.

---

## 8. 알려진 패턴 & 주의사항

- **Form 첨부 저장**: ThisItem + Parent.Default 조합 필수. Items = Parent.Default.
- **보고서(Analysis_Report_DB)**: 의뢰 첨부와 완전 분리. frmReportUpload는 SubmitForm만, OnSuccess에서 후처리.
- **인쇄(PrintScreen)**: HtmlViewer 인라인 스타일만 사용(class 불안정). Teams에서 data:URI Launch 차단됨 → HtmlText + Print() 방식.
- **타이밍**: SubmitForm → OnSuccess 순서. Refresh 후 LookUp 해야 최신값. 후처리는 OnSuccess에 몰아넣어 타이밍 꼬임 방지.
- **galRevisionHistory에서 Rev별 첨부 조회**: 보류 상태. 추후 별도 화면으로 구성 예정.

---

## 9. 작업 컨벤션

- 수식 앞에 `=` 붙이지 않기 (복붙용)
- 코드 줄 때 적용 위치 명시 (컨트롤 + 속성 + 스크린)
- 수식 아래 "→ 이 수식이 하는 일" 한 줄 요약
- 메일 발신자 인사말: "안녕하십니까, 김은성 주임 입니다"로 시작
- 변경사항은 GitHub로 버전 관리, *.msapp는 .gitignore

---

## 10. 최근 변경 이력

- RequestNo 포맷 변경: `AR-2026-001` → `CAE26033` (CAE + 연도2 + 순번3, 33번부터)
- btnSaveWorkProgress: 진행 정보 저장 시 의뢰자 이메일 알림 추가
- **Revision별 첨부파일 분리 구현** (frmRevisionAttach 체이닝, frmPrevRevAttachView 이전 버전 참고, 좌우 레이아웃)
- Analysis_Request_Revision_DB 커스텀 컬럼 Required 전체 해제

### 보류 / 다음 작업 후보
- galRevisionHistory Rev 클릭 시 첨부 조회 화면 (별도 화면 구성 예정)
