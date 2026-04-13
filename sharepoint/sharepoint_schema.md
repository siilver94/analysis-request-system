# SharePoint Schema – Analysis Request System

본 문서는 해석 의뢰 현황판 시스템에서 사용하는 SharePoint 데이터 구조의 최신 기준을 정의합니다.  
본 시스템은 **Power Apps + SharePoint** 기반으로 동작하며, 현재 아래 저장소를 사용합니다.

---

# 전체 구조

## Lists
- `Analysis_Request_DB` : 메인 의뢰 리스트
- `Analysis_Request_Revision_DB` : 수정 이력 리스트
- `Analysis_Request_Log_DB` : 처리 로그 리스트

## Document Libraries
- `Analysis_Request_Files` : 의뢰 첨부 / 수정 첨부 / 작업 참고자료
- `Analysis_Report_Files` : 최종 해석 보고서

---

# 1. Analysis_Request_DB

## 설명
현재 기준의 **최신 해석 의뢰 정보**를 저장하는 메인 리스트입니다.  
현황판, 상세 조회, 승인/반려, 진행 상태 표시의 기준 데이터입니다.

## 컬럼 정의

| Column Name | Type | Required | Description |
|---|---|---|---|
| Title | Single line text | No | 해석명(제목) |
| RequestNo | Single line text | No | 시스템 생성 의뢰번호 (예: AR-2026-0001) |
| RequestIDKey | Single line text | No | 시스템 생성 내부 연결 키 |
| DepartmentName | Single line text | No | 부서명 |
| RequesterName | Single line text | No | 의뢰자 이름 |
| RequesterEmail | Single line text | No | 의뢰자 이메일 |
| ProductName | Single line text | No | 품명 |
| PartNo | Single line text | No | 품번 |
| AnalysisReason | Multiple line text | No | 해석 사유 |
| AnalysisCondition | Multiple line text | No | 해석 조건 및 검토사항 |
| RequesterLeaderEmail | Single line text | No | 의뢰자 팀장 이메일 |
| VAApprover1Email | Single line text | No | 가상검증 승인자 1 이메일 |
| VAApprover2Email | Single line text | No | 가상검증 승인자 2 이메일 |
| Status | Choice | No | 현재 상태 |
| AssigneeEmail | Single line text | No | 담당자 이메일 |
| ProgressPct | Number | No | 진행률(0~100) |
| ExpectedFinishDate | Date and Time | No | 예상 완료일 |
| WorkerMemo | Multiple line text | No | 작업자 메모 |
| LastRevisionNo | Number | No | 최신 수정 번호 |
| LastRequesterModifiedAt | Date and Time | No | 마지막 의뢰 수정일 |
| AssigneeLastReadRevisionNo | Number | No | 담당자가 마지막 확인한 수정 번호 |
| HasUnreadRevision | Yes/No | No | 새 수정 도착 여부 |
| FinalReportAttached | Yes/No | No | 최종 보고서 첨부 여부 |
| FinalReportFilePath | Hyperlink or Single line text | No | 최종 보고서 경로 |
| SubmittedAt | Date and Time | No | 최초 등록일 |
| CompletedAt | Date and Time | No | 완료일 |
| RejectedAt | Date and Time | No | 반려일 |
| RejectReason | Multiple line text | No | 반려 사유 |
| IsActive | Yes/No | No | 활성 여부 |

## Status 선택값

- `PendingTeamLeader`
- `PendingVAApproval`
- `InProgress`
- `Completed`
- `Rejected`

## 설계 원칙

- SharePoint의 Required는 대부분 **No**로 유지합니다.
- 필수 입력 검증은 **Power Apps 버튼 로직에서 수행**합니다.
- `RequestNo`, `RequestIDKey`, `Status` 등 시스템 자동 생성/입력 컬럼은 SharePoint 필수로 두지 않습니다.

---

# 2. Analysis_Request_Revision_DB

## 설명
의뢰 수정 이력을 버전별로 저장하는 리스트입니다.  
메인 리스트에는 최신값만 남기고, 이전 버전은 이 리스트에 저장합니다.

## 컬럼 정의

| Column Name | Type | Required | Description |
|---|---|---|---|
| Title | Single line text | No | 예: AR-2026-0001 Rev.2 |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| RequestNo | Single line text | No | 의뢰번호 |
| RevisionNo | Number | No | 수정 번호 |
| RevisedByName | Single line text | No | 수정자 이름 |
| RevisedByEmail | Single line text | No | 수정자 이메일 |
| RevisedAt | Date and Time | No | 수정 시각 |
| DepartmentNameSnapshot | Single line text | No | 수정 당시 부서명 |
| RequesterNameSnapshot | Single line text | No | 수정 당시 의뢰자 |
| AnalysisTitleSnapshot | Single line text | No | 수정 당시 해석명 |
| ProductNameSnapshot | Single line text | No | 수정 당시 품명 |
| PartNoSnapshot | Single line text | No | 수정 당시 품번 |
| AnalysisReasonSnapshot | Multiple line text | No | 수정 당시 해석 사유 |
| AnalysisConditionSnapshot | Multiple line text | No | 수정 당시 해석 조건 및 검토사항 |
| FileFolderPath | Hyperlink or Single line text | No | 해당 수정본 첨부 폴더 경로 |
| RevisionComment | Multiple line text | No | 수정 메모 |

---

# 3. Analysis_Request_Log_DB

## 설명
의뢰 처리 이력을 저장하는 로그 리스트입니다.  
등록, 수정, 승인, 반려, 담당자 배정, 완료 등의 시스템/사용자 행동을 기록합니다.

## 컬럼 정의

| Column Name | Type | Required | Description |
|---|---|---|---|
| Title | Single line text | No | 로그 제목 |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| RequestNo | Single line text | No | 의뢰번호 |
| LogType | Choice | No | 로그 유형 |
| LogMessage | Multiple line text | No | 화면 표시용 메시지 |
| ActionByName | Single line text | No | 수행자 이름 |
| ActionByEmail | Single line text | No | 수행자 이메일 |
| ActionRole | Single line text | No | 수행 역할 |
| ActionAt | Date and Time | No | 수행 시각 |
| RelatedRevisionNo | Number | No | 연관 수정 번호 |

## LogType 선택값

- `Submit`
- `Modify`
- `TeamLeaderApprove`
- `TeamLeaderReject`
- `VAApprove`
- `VAReject`
- `Assign`
- `ProgressUpdate`
- `Complete`
- `FinalReportUpload`

## Power Apps Patch 주의사항

`Status`, `LogType`은 SharePoint **Choice 컬럼**이므로 Patch 시 문자열이 아니라 아래 형식으로 넣어야 합니다.

```powerfx
Status: { Value: "PendingTeamLeader" }
LogType: { Value: "Submit" }
