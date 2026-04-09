# SharePoint Schema – Analysis Request System

본 문서는 해석 의뢰 현황판 시스템에서 사용하는 SharePoint 데이터 구조를 정의합니다.  
본 시스템은 Power Apps + SharePoint 기반으로 동작하며, 다음과 같은 저장소로 구성됩니다.

---

# 📌 전체 구조

## Lists
- Analysis_Request_DB (메인 의뢰)
- Analysis_Request_Revision_DB (수정 이력)
- Analysis_Request_Log_DB (로그)

## Document Libraries
- Analysis_Request_Files (의뢰 및 수정 첨부파일)
- Analysis_Report_Files (최종 해석 보고서)

---

# 1. Analysis_Request_DB (메인 리스트)

## 설명
현재 기준의 최신 의뢰 상태를 저장하는 메인 테이블

## 컬럼 정의

| Column Name | Type | Required | Description |
|---|---|---|---|
| Title | Single line text | O | 해석명 (제목) |
| RequestNo | Single line text | O | 의뢰번호 |
| RequestIDKey | Single line text | O | 내부 고유 키 |
| DepartmentName | Single line text | O | 부서명 |
| RequesterName | Single line text | O | 의뢰자 이름 |
| RequesterEmail | Single line text | O | 의뢰자 이메일 |
| ProductName | Single line text | O | 품명 |
| PartNo | Single line text | O | 품번 |
| AnalysisReason | Multiple line text | O | 해석 사유 |
| AnalysisCondition | Multiple line text | O | 해석 조건 및 검토사항 |
| Status | Choice | O | 상태값 |
| AssigneeName | Single line text |  | 담당자 이름 |
| AssigneeEmail | Single line text |  | 담당자 이메일 |
| ProgressPct | Number |  | 진행률 |
| ExpectedFinishDate | Date |  | 예상 완료일 |
| WorkerMemo | Multiple line text |  | 작업자 메모 |
| LastRevisionNo | Number | O | 최신 수정 번호 |
| LastRequesterModifiedAt | DateTime |  | 마지막 수정일 |
| AssigneeLastReadRevisionNo | Number |  | 담당자 확인한 수정번호 |
| HasUnreadRevision | Yes/No |  | 수정 알림 여부 |
| FinalReportAttached | Yes/No |  | 보고서 첨부 여부 |
| FinalReportFilePath | Hyperlink |  | 보고서 경로 |
| SubmittedAt | DateTime | O | 등록일 |
| CompletedAt | DateTime |  | 완료일 |
| RejectedAt | DateTime |  | 반려일 |
| RejectReason | Multiple line text |  | 반려 사유 |

---

## 상태값 (Status Choice)

- PendingTeamLeader
- PendingVAApproval
- InProgress
- Completed
- Rejected

---

# 2. Analysis_Request_Revision_DB (수정 이력)

## 설명
의뢰 수정 이력을 버전별로 저장

## 컬럼 정의

| Column Name | Type | Required | Description |
|---|---|---|---|
| Title | Single line text | O | 예: AR-2026-0001 Rev.2 |
| RequestIDKey | Single line text | O | 메인 연결 키 |
| RequestNo | Single line text | O | 의뢰번호 |
| RevisionNo | Number | O | 수정 번호 |
| RevisedByName | Single line text | O | 수정자 이름 |
| RevisedByEmail | Single line text | O | 수정자 이메일 |
| RevisedAt | DateTime | O | 수정 일시 |
| DepartmentNameSnapshot | Single line text | O | 당시 부서명 |
| RequesterNameSnapshot | Single line text | O | 당시 의뢰자 |
| AnalysisTitleSnapshot | Single line text | O | 당시 제목 |
| ProductNameSnapshot | Single line text | O | 당시 품명 |
| PartNoSnapshot | Single line text | O | 당시 품번 |
| AnalysisReasonSnapshot | Multiple line text | O | 당시 해석 사유 |
| AnalysisConditionSnapshot | Multiple line text | O | 당시 조건 |
| FileFolderPath | Hyperlink |  | 첨부 폴더 경로 |
| RevisionComment | Multiple line text |  | 수정 메모 |

---

# 3. Analysis_Request_Log_DB (로그)

## 설명
의뢰 처리 흐름 기록 (이력 추적용)

## 컬럼 정의

| Column Name | Type | Required | Description |
|---|---|---|---|
| Title | Single line text | O | 로그 제목 |
| RequestIDKey | Single line text | O | 연결 키 |
| RequestNo | Single line text | O | 의뢰번호 |
| LogType | Choice | O | 로그 유형 |
| LogMessage | Multiple line text | O | 표시 메시지 |
| ActionByName | Single line text | O | 수행자 |
| ActionByEmail | Single line text | O | 이메일 |
| ActionRole | Single line text | O | 역할 |
| ActionAt | DateTime | O | 수행 시각 |
| RelatedRevisionNo | Number |  | 연관 수정번호 |

---

## LogType 값

- Submit
- Modify
- TeamLeaderApprove
- TeamLeaderReject
- VAApprove
- VAReject
- Assign
- ProgressUpdate
- Complete
- FinalReportUpload

---

# 4. Analysis_Request_Files (문서 라이브러리)

## 설명
의뢰 첨부 및 수정 파일 저장

## 폴더 구조

Analysis_Request_Files/
└─ AR-2026-0001/
├─ Rev-0/
├─ Rev-1/
├─ Rev-2/
└─ Worker/


---

## 메타데이터 컬럼

| Column Name | Type | Description |
|---|---|---|
| RequestIDKey | Single line text | 의뢰 연결 키 |
| RequestNo | Single line text | 의뢰번호 |
| RevisionNo | Number | 수정 번호 |
| FileCategory | Choice | Request / Revision / Worker |
| UploadedByName | Single line text | 업로더 |
| UploadedByEmail | Single line text | 이메일 |
| UploadedAt | DateTime | 업로드 시각 |

---

# 5. Analysis_Report_Files (문서 라이브러리)

## 설명
최종 해석 보고서 저장

## 구조

Analysis_Report_Files/
└─ AR-2026-0001/
└─ final-report.pdf


---

## 메타데이터 컬럼

| Column Name | Type | Description |
|---|---|---|
| RequestIDKey | Single line text | 의뢰 연결 키 |
| RequestNo | Single line text | 의뢰번호 |
| UploadedByName | Single line text | 담당자 |
| UploadedByEmail | Single line text | 이메일 |
| UploadedAt | DateTime | 업로드 시각 |

---

# 📌 설계 원칙

- 메인 리스트는 **최신 데이터만 유지**
- 수정 이력은 별도 리스트에서 관리
- 첨부파일은 리스트가 아닌 **문서 라이브러리 사용**
- 리스트 간 연결은 Lookup 대신 **Key 기반(RequestIDKey)** 사용
- Power Apps 성능을 고려한 구조

---

# 📌 향후 확장

- 출력 기능 (Print Template)
- 알림 로그 고도화
- KPI 분석 연계
- 파일 버전 자동 관리

