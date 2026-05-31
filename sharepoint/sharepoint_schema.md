# SharePoint List 스키마 정의

**프로젝트**: 연구소 해석 의뢰 현황판  
**플랫폼**: Microsoft 365 (무료 플랜)  
**마지막 업데이트**: 2026-05-31

---

## 데이터소스 구성

| 리스트명 | 용도 | 비고 |
|---------|------|------|
| Analysis_Request_DB | 메인 의뢰 데이터 | 핵심 |
| Analysis_Request_Revision_DB | 수정 이력 (버전 스냅샷) | |
| Analysis_Request_Log_DB | 상태 변경 로그 | 타임라인 조회용 |
| Analysis_Report_DB | 해석 보고서 전용 | 의뢰 첨부와 분리 |
| UserInfo_DB | 사용자 정보 | 역할 판단용 |

미사용 (연결만 되어있음):
- Analysis_Request_Files (Document Library)
- Analysis_Report_Files (Document Library)

---

## 1. Analysis_Request_DB

메인 의뢰 데이터 리스트.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공 |
| RequestNo | Single line text | No | 의뢰번호 (예: REQ-2026-001) |
| RequestIDKey | Single line text | No | 고유 키 (GUID 기반) |
| Status | Choice | No | 아래 선택지 참고 |
| RequesterName | Single line text | No | 등록자 이름 |
| RequesterEmail | Single line text | No | 등록자 이메일 |
| RequesterLeaderEmail | Single line text | No | 등록자 팀장 이메일 |
| RequesterTeam | Single line text | No | 등록 시점 팀명 스냅샷 ★ 추가됨 |
| DepartmentName | Single line text | No | 부문명 |
| ProductName | Single line text | No | 품명 |
| PartNo | Single line text | No | 품번 |
| AnalysisTitle | Single line text | No | 해석 제목 |
| AnalysisReason | Multiple lines of text | No | 해석 사유 |
| AnalysisCondition | Multiple lines of text | No | 해석 조건 및 검토사항 |
| SubmittedAt | Date and Time | No | 등록(제출) 일시 |
| AssigneeEmail | Single line text | No | 담당자 이메일 (VA 승인 시 세팅) |
| AssigneeName | Single line text | No | 담당자 이름 |
| ProgressPct | Number | No | 진행률 0~100 |
| ExpectedFinishDate | Date and Time | No | 예상 완료일 |
| WorkerMemo | Multiple lines of text | No | 작업자 메모 |
| CompletedAt | Date and Time | No | 완료 일시 |
| RejectedAt | Date and Time | No | 반려 일시 |
| RejectReason | Multiple lines of text | No | 반려 사유 |
| LastRevisionNo | Number | No | 최신 수정 버전 번호 |
| HasUnreadRevision | Yes/No | No | 담당자 미확인 수정 여부 (칸반 뱃지용) |
| HasRequestAttachment | Yes/No | No | 의뢰 첨부파일 여부 |
| FinalReportAttached | Yes/No | No | 보고서 첨부 여부 ★ 추가됨 |
| 첨부 파일 | Attachments | - | SharePoint 기본 제공 (의뢰 첨부 전용) |

**Status 선택지:**

    PendingTeamLeader   팀장 승인 대기
    PendingVAApproval   가상검증팀 승인 대기
    InProgress          진행중
    Completed           완료
    Rejected            반려

---

## 2. Analysis_Request_Revision_DB

수정 이력 리스트. 의뢰 수정 시 버전별 스냅샷 저장.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공 |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| RevisionNo | Number | No | 수정 버전 번호 (1, 2, 3...) |
| RevisionComment | Multiple lines of text | No | 수정 사유 |
| RevisedByName | Single line text | No | 수정자 이름 |
| RevisedByEmail | Single line text | No | 수정자 이메일 |
| RevisedAt | Date and Time | No | 수정 일시 |
| ProductNameSnapshot | Single line text | No | 수정 시점 품명 스냅샷 |
| PartNoSnapshot | Single line text | No | 수정 시점 품번 스냅샷 |
| AnalysisTitleSnapshot | Single line text | No | 수정 시점 해석제목 스냅샷 |
| 첨부 파일 | Attachments | - | SharePoint 기본 제공 |

---

## 3. Analysis_Request_Log_DB

상태 변경 로그 리스트. 처리 현황 타임라인 날짜 조회용.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공 |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| LogType | Choice | No | 아래 선택지 참고 |
| ActionAt | Date and Time | No | 처리 일시 |
| ActorEmail | Single line text | No | 처리자 이메일 |
| ActorName | Single line text | No | 처리자 이름 |
| Memo | Multiple lines of text | No | 비고 |

**LogType 선택지:**

    Submit              등록
    TeamLeaderApprove   팀장 승인
    TeamLeaderReject    팀장 반려
    VAApprove           VA 승인
    VAReject            VA 반려
    Complete            작업 완료

**조회 패턴 (타임라인용):**

    LookUp(
        Analysis_Request_Log_DB,
        RequestIDKey = varSelectedRequest.RequestIDKey &&
        LogType.Value = "TeamLeaderApprove"
    ).ActionAt

---

## 4. Analysis_Report_DB ★ 신규 생성

해석 보고서 전용 리스트. Analysis_Request_DB 기본 첨부파일과 분리.

> **분리 배경**: SharePoint List는 첨부파일 컬럼이 1개뿐이라 의뢰 첨부와 보고서 첨부를 같은 공간에서 분리할 수 없음. 별도 List 생성으로 해결.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | "{RequestNo}_보고서" |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| RequestNo | Single line text | No | 의뢰번호 |
| UploadedByName | Single line text | No | 업로드자 이름 |
| UploadedByEmail | Single line text | No | 업로드자 이메일 |
| UploadedAt | Date and Time | No | 업로드 일시 |
| 첨부 파일 | Attachments | - | SharePoint 기본 제공 (보고서 전용) |

**연동 방식:**

    // 해당 의뢰의 보고서 레코드 조회
    LookUp(
        Analysis_Report_DB,
        RequestIDKey = varSelectedRequest.RequestIDKey
    )

    // 보고서 첨부파일 조회 (DataCardValue1.Items)
    LookUp(
        Analysis_Report_DB,
        RequestIDKey = varSelectedRequest.RequestIDKey
    ).'첨부 파일'

**주의사항:**

- Document Library가 아닌 일반 **List**로 생성해야 PowerApps Form과 연동 가능
- FinalReportAttached 컬럼(Analysis_Request_DB)은 이 List의 첨부파일 수 기준으로 동기화

---

## 5. UserInfo_DB

사용자 정보 리스트. 앱 시작 시 현재 사용자 정보 조회용.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공 |
| 이름_한글 | Single line text | No | 한글 이름 |
| e-mail | Single line text | No | 이메일 (소문자로 저장 권장) |
| 부문 | Single line text | No | 부문명 |
| 팀 | Single line text | No | 팀명 |
| 직급2 | Single line text | No | 직급 (팀장, 소장, 이사 등) |
| IsVAApprover | Yes/No | No | VA 승인자 여부 ★ 개선 예정 |

**역할 판단 로직 (앱 OnStart):**

    // 현재 사용자 정보 조회
    Set(
        varUserRecord,
        LookUp(UserInfo_DB, Lower('e-mail') = Lower(User().Email))
    );
    Set(varMyName,      varUserRecord.이름_한글);
    Set(varMyEmail,     Lower(User().Email));
    Set(varMyDept,      varUserRecord.부문);
    Set(varMyTeam,      varUserRecord.팀);
    Set(varMyJobGrade,  varUserRecord.직급2);
    Set(varIsTeamLeader, varMyJobGrade in ["팀장", "소장", "이사"]);

    // VA 승인자 판단 (현재 이메일 하드코딩 — 추후 IsVAApprover 컬럼으로 개선 예정)
    Set(
        varIsVAApprover,
        varMyEmail in [
            Lower("donggyu.lee@tym.world"),
            Lower("E2603003@tym.world")
        ]
    );

---

## 리스트 간 연결 구조

    Analysis_Request_DB (RequestIDKey)
        ├── Analysis_Request_Revision_DB (RequestIDKey)
        ├── Analysis_Request_Log_DB (RequestIDKey)
        └── Analysis_Report_DB (RequestIDKey)

    UserInfo_DB (e-mail)
        └── Analysis_Request_DB 의 RequesterEmail, AssigneeEmail과 대조

---

## 개선 예정 사항

1. **VA 승인자 관리**: 현재 이메일 2개 하드코딩 → UserInfo_DB.IsVAApprover 컬럼으로 동적 관리
2. **Document Library 활용 여부**: Analysis_Request_Files, Analysis_Report_Files 현재 미사용. 추후 검토.

---

## 알려진 제약 및 주의사항

- SharePoint List는 첨부파일 컬럼이 기본 1개뿐 → 용도별 분리 시 별도 List 필요
- Document Library는 PowerApps Form의 Attachments 컨트롤과 직접 연동 불가
- Analysis_Report_DB는 반드시 **일반 List**로 생성 (Document Library 아님)
- Delegation 주의: Filter + Choice 필드 조합은 delegation 가능하나 복잡한 중첩 조건 시 주의
- LookUp으로 타임라인 날짜 조회 시 LogType.Value (Choice 컬럼) 사용 → delegation 가능
