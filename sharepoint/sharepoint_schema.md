# SharePoint List 스키마 정의

**프로젝트**: 연구소 해석 의뢰 현황판
**플랫폼**: Microsoft 365 (무료 플랜)
**마지막 업데이트**: 2026-08-05

---

## 데이터소스 구성

| 리스트명 | 용도 | 비고 |
|---------|------|------|
| Analysis_Request_DB | 메인 의뢰 데이터 | 핵심 |
| Analysis_Request_Revision_DB | 수정 이력 + Revision별 첨부파일 | 첨부 분리 저장 |
| Analysis_Request_Log_DB | 상태 변경 로그 | 타임라인 조회용 |
| Analysis_Report_DB | 해석 보고서 전용 | 의뢰 첨부와 분리 |
| UserInfo_DB | 사용자 정보 | 역할 판단용 |
| Project_DB | 업무_계정·프로젝트명 조회 전용 (신규) | **읽기전용 참조**. "중앙기술연구소 DB" 사이트의 공용 리스트, 이 앱은 조회만 함 |

미사용 (연결만 되어있음):
- Analysis_Request_Files (Document Library)
- Analysis_Report_Files (Document Library)

---

## 1. Analysis_Request_DB

메인 의뢰 데이터 리스트. 항상 **현재(최신) 상태**를 보관.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공. 해석명(제목) 저장 |
| RequestNo | Single line text | No | 의뢰번호 (예: CAE26033) |
| RequestIDKey | Single line text | No | 고유 키 (GUID 기반) |
| Status | Choice | No | 아래 선택지 참고 |
| RequesterName | Single line text | No | 등록자 이름 |
| RequesterEmail | Single line text | No | 등록자 이메일 |
| RequesterLeaderEmail | Single line text | No | 등록자 팀장 이메일. 등록 시점에 UserInfo_DB에서 조회해 저장 — 완료 알림 메일 발송 시에도 재사용(재조회 안 함) |
| RequesterTeam | Single line text | No | 등록 시점 팀명 스냅샷 |
| DepartmentName | Single line text | No | 부문명 |
| ProductName | Single line text | No | 품명 |
| PartNo | Single line text | No | 품번 |
| WorkCategory | Single line text | No | **(신규)** 업무_계정. Project_DB 기준 "제품개발/기술개발/설계변경/원가절감" 중 선택 — 앱에서 필수 입력으로 검증(SharePoint 상은 Required 해제, 코드에서 직접 Patch하기 때문) |
| ProjectName | Single line text | No | **(신규)** 프로젝트명. Project_DB에서 WorkCategory에 종속되어 선택 — 앱에서 필수 입력으로 검증 |
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
| LastRevisionNo | Number | No | 최신 Revision 번호. 최초 등록 시 1부터 시작 |
| AssigneeLastReadRevisionNo | Number | No | 담당자가 마지막으로 확인한 Revision 번호 |
| HasUnreadRevision | Yes/No | No | 담당자 미확인 수정 여부 (칸반 뱃지용) |
| FinalReportAttached | Yes/No | No | 보고서 첨부 여부 (Analysis_Report_DB와 동기화) |
| 첨부 파일 | Attachments | - | SharePoint 기본 제공. ⚠️ 현재 미사용 (의뢰 첨부는 Revision DB로 이동) |

> **해석명 저장 위치**: 폼에서 해석명은 기본 `Title` 필드에 저장됨 (display name "제목"). 코드에서는 `varSelectedRequest.제목`으로 접근.
>
> **첨부파일 변경 (중요)**: 의뢰 첨부파일은 이제 `Analysis_Request_Revision_DB`에 Revision별로 저장됨. 이 List의 `첨부 파일` 컬럼은 더 이상 쓰지 않음 (frmAnalysisRequest의 첨부 DataCard Visible=false 처리).
>
> **WorkCategory/ProjectName (신규)**: 값의 원본은 Project_DB지만, 이 두 컬럼엔 선택된 텍스트값만 복사해서 저장(참조 키 아님). Project_DB에서 프로젝트명이 나중에 바뀌어도 과거 저장된 값은 그대로 유지됨(스냅샷 성격).

**Status 선택지:**

    PendingTeamLeader   팀장 승인 대기
    PendingVAApproval   가상검증팀 승인 대기
    InProgress          진행중
    Completed           완료
    Rejected            반려

---

## 2. Analysis_Request_Revision_DB

수정 이력 리스트. **각 Revision = 그 시점 저장된 버전(텍스트 스냅샷 + 첨부파일)**.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공. "{RequestNo} Rev.{n}" |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| RequestNo | Single line text | No | 의뢰번호 |
| RevisionNo | Number | No | Revision 번호 (1, 2, 3...) |
| RevisionComment | Multiple lines of text | No | 수정 사유 (최초 등록 시 빈 값) |
| RevisedByName | Single line text | No | 작성/수정자 이름 |
| RevisedByEmail | Single line text | No | 작성/수정자 이메일 |
| RevisedAt | Date and Time | No | 작성/수정 일시 |
| AnalysisTitleSnapshot | Single line text | No | 해당 버전 해석명 스냅샷 |
| ProductNameSnapshot | Single line text | No | 해당 버전 품명 스냅샷 |
| PartNoSnapshot | Single line text | No | 해당 버전 품번 스냅샷 |
| WorkCategorySnapshot | Single line text | No | **(신규)** 해당 버전 업무_계정 스냅샷 |
| ProjectNameSnapshot | Single line text | No | **(신규)** 해당 버전 프로젝트명 스냅샷 |
| AnalysisReasonSnapshot | Multiple lines of text | No | 해당 버전 해석사유 스냅샷 |
| AnalysisConditionSnapshot | Multiple lines of text | No | 해당 버전 해석조건 스냅샷 |
| DepartmentNameSnapshot | Single line text | No | 해당 버전 부문 스냅샷 |
| RequesterNameSnapshot | Single line text | No | 해당 버전 작성자 스냅샷 |
| 첨부 파일 | Attachments | - | SharePoint 기본 제공. **Revision별 첨부파일 (활성 사용)** |

> **Revision 모델 (Rev별 첨부 분리)**:
> - 최초 등록 → Rev.1 생성 (원본 텍스트 + 첨부 직접 업로드)
> - 수정 → Rev.N 생성 (새 텍스트 + 새 첨부, "새로 올리기" 방식)
> - 현재/최신 = 가장 높은 RevisionNo
> - **"새로 올리기"**: 수정 시 이전 파일이 자동으로 넘어오지 않음. 유지하려면 다시 업로드. (SharePoint List 간 첨부 복사가 PowerApps에서 불가하기 때문)
>
> **Required 설정**: 위 커스텀 컬럼은 전부 SharePoint에서 **필수(Required) 해제** 상태여야 함 (WorkCategorySnapshot/ProjectNameSnapshot 포함). 첨부 폼(frmRevisionAttach)이 첨부만 제출하고 나머지 컬럼은 OnSuccess에서 Patch하기 때문. Title(기본 필드)은 건드리지 않음.

---

## 3. Analysis_Request_Log_DB

상태 변경 로그 리스트. 처리 현황 타임라인 날짜 조회용.

| 컬럼명 | 타입 | 필수 | 비고 |
|-------|------|------|------|
| Title | Single line text | No | SharePoint 기본 제공. LogType 값 저장 |
| RequestIDKey | Single line text | No | 메인 의뢰 연결 키 |
| RequestNo | Single line text | No | 의뢰번호 |
| LogType | Choice | No | 아래 선택지 참고 |
| LogMessage | Multiple lines of text | No | 로그 메시지 (사람이 읽는 설명) |
| ActionByName | Single line text | No | 처리자 이름 |
| ActionByEmail | Single line text | No | 처리자 이메일 |
| ActionRole | Single line text | No | 처리자 역할 (Requester, RequesterLeader, VAApprover, Assignee) |
| ActionAt | Date and Time | No | 처리 일시 |
| RelatedRevisionNo | Number | No | 관련 Revision 번호 |

**LogType 선택지 (⚠️ 이전 세션에서 추가됐으나 이 문서에 반영이 늦었던 항목 포함):**

    Submit              등록
    Modify              수정
    TeamLeaderApprove   팀장 승인
    TeamLeaderReject    팀장 반려
    VAApprove           VA 승인
    VAReject            VA 반려
    Assign              담당자 배정
    ProgressUpdate      진행 정보 업데이트
    Complete            작업 완료
    AssigneeChanged     담당자 변경 (진행 중, 박윤주 팀장 전용)
    TitleChanged        해석명 변경 (가상검증팀)

> **주의**: 새 LogType 값을 코드에서 Patch하기 전에, SharePoint에서 이 Choice 컬럼의 선택지에 해당 값을 먼저 추가해야 합니다. 선택지에 없는 값을 Patch하면 에러가 발생할 수 있습니다.

**조회 패턴 (타임라인용):**

    LookUp(
        Analysis_Request_Log_DB,
        RequestIDKey = varSelectedRequest.RequestIDKey &&
        LogType.Value = "TeamLeaderApprove"
    ).ActionAt

---

## 4. Analysis_Report_DB

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

    // 보고서 첨부파일 조회
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
| IsVAApprover | Yes/No | No | VA 승인자 여부 (개선 예정 — 현재 미사용) |

**역할 판단 로직 (WorkRequest_Screen OnVisible):**

    // 현재 사용자 정보 조회
    Set(varMyEmail, Lower(User().Email));
    Set(
        varMyInfo,
        LookUp(UserInfo_DB, Lower('e-mail') = varMyEmail)
    );
    Set(varMyName,     Coalesce(varMyInfo.이름_한글, /* User().FullName 파싱 fallback */ ""));
    Set(varMyDept,     Coalesce(varMyInfo.부문, ""));
    Set(varMyTeam,     Coalesce(varMyInfo.팀, ""));
    Set(varMyJobGrade, Coalesce(varMyInfo.직급2, ""));

    // 결재 권한자 판단
    Set(varIsTeamLeader, varMyJobGrade in ["팀장", "소장", "이사"]);

    // VA 승인자 판단 (현재 이메일 하드코딩 — 추후 IsVAApprover 컬럼으로 개선 예정)
    Set(
        varIsVAApprover,
        varMyEmail in [
            Lower("yoonju.park@tym.world"),
            Lower("donghyeon.gim@tym.world")
        ]
    );
    Set(varVAApprover1Email, Lower("yoonju.park@tym.world"));
    Set(varVAApprover2Email, Lower("donghyeon.gim@tym.world"));

---

## 6. Project_DB (신규, 읽기전용 참조)

"중앙기술연구소 DB" 사이트에 있는 공용 리스트. 이 앱은 **조회만** 하며 데이터를 쓰지 않음(다른 앱/용도와 공유하는 리스트라 스키마 변경 불가).

| 컬럼명(실사용) | 비고 |
|-------|------|
| 업무_계정 | 대분류. 실제 리스트엔 다양한 값이 있지만, 이 앱에서는 **"제품개발", "기술개발", "설계변경", "원가절감" 4개만** 골라서 사용 (코드에 하드코딩된 고정 목록) |
| 프로젝트명 | 중분류. 위 4개 업무_계정에 속한 것만 Filter해서 사용 |

> 그 외 컬럼(PM, 관련기종명, 시작/종료날짜, 금액 등 다수)은 이 앱에서 사용하지 않음.

**연동 방식 (WorkRequest_Screen.OnVisible에서 캐싱):**

    ClearCollect(
        colWorkCategoryOptions,
        {WorkCategoryValue: "제품개발"},
        {WorkCategoryValue: "기술개발"},
        {WorkCategoryValue: "설계변경"},
        {WorkCategoryValue: "원가절감"}
    );

    ClearCollect(
        colProjectList,
        RenameColumns(
            Filter(
                Project_DB,
                업무_계정 in ["제품개발", "기술개발", "설계변경", "원가절감"]
            ),
            업무_계정, WorkCategoryValue,
            프로젝트명, ProjectNameValue
        )
    );

> **업무_계정은 코드에 하드코딩**되어 있어서, Project_DB에 새로운 업무_계정 구분이 생겨도 이 앱 드롭다운엔 자동으로 반영되지 않음 (의도된 동작 — "제품개발/기술개발/설계변경/원가절감" 외엔 이 앱과 무관한 프로젝트로 간주). 새 구분을 추가하고 싶으면 앱 코드의 하드코딩 목록도 같이 수정해야 함.

---

## 리스트 간 연결 구조

    Analysis_Request_DB (RequestIDKey)
        ├── Analysis_Request_Revision_DB (RequestIDKey)   ← 수정 이력 + Revision별 첨부
        ├── Analysis_Request_Log_DB (RequestIDKey)        ← 처리 로그
        └── Analysis_Report_DB (RequestIDKey)             ← 보고서 첨부

    UserInfo_DB (e-mail)
        └── Analysis_Request_DB 의 RequesterEmail, AssigneeEmail과 대조

    Project_DB (업무_계정, 프로젝트명)
        └── Analysis_Request_DB 의 WorkCategory, ProjectName에 텍스트값만 복사 저장
            (RequestIDKey 같은 키 연결 아님 — 참조 후 스냅샷하는 방식, Project_DB 값이 바뀌어도 과거 기록엔 영향 없음)

---

## 개선 예정 사항

1. **VA 승인자 관리**: 현재 이메일 2개 하드코딩 → UserInfo_DB.IsVAApprover 컬럼으로 동적 관리
2. **Revision 첨부 조회 화면**: galRevisionHistory에서 Rev 클릭 시 해당 Rev 첨부 조회 → 별도 화면으로 구성 예정
3. **Document Library 활용 여부**: Analysis_Request_Files, Analysis_Report_Files 현재 미사용. 추후 검토
4. **업무_계정 고정 목록 관리 (신규)**: 현재 "제품개발/기술개발/설계변경/원가절감" 4개가 코드에 하드코딩됨 → 추후 항목이 자주 바뀔 것 같으면 별도 설정용 컬렉션/리스트로 분리하는 것도 고려 가능(현재는 4개 고정이라 낮은 우선순위)

---

## 알려진 제약 및 주의사항

- SharePoint List는 첨부파일 컬럼이 기본 1개뿐 → 용도별 분리 시 별도 List 필요
- **이미 저장된 SharePoint 첨부는 List 간 복사 불가** (PowerApps 한계). 업로드 순간엔 메모리에 실체가 있어 Form으로 저장 가능 → Revision별 첨부는 "새로 올리기" 방식 채택
- 첨부파일은 반드시 Form을 통해 저장 (Patch로 직접 첨부 불가)
- Document Library는 PowerApps Form의 Attachments 컨트롤과 직접 연동 불가
- Analysis_Report_DB / Analysis_Request_Revision_DB는 반드시 **일반 List**로 생성 (Document Library 아님)
- Delegation 주의: Filter + Choice 필드 조합은 delegation 가능하나 복잡한 중첩 조건 시 주의
- LookUp으로 타임라인 날짜 조회 시 LogType.Value (Choice 컬럼) 사용 → delegation 가능
- **Project_DB는 공용 리스트라 이 앱 마음대로 스키마 변경 불가** — 컬럼 추가/이름 변경 필요하면 관리 주체와 먼저 협의 필요
