# 해석 의뢰 현황판 (Analysis Request System)

TYM 중앙기술연구소의 해석(CAE) 의뢰를 등록·결재·진행·완료까지 관리하는 사내 업무 앱입니다.
Microsoft Teams 탭으로 배포되는 PowerApps Canvas App이며, 데이터는 전적으로 SharePoint List를 사용합니다.

---

## 개요

| 항목 | 내용 |
|--|--|
| 플랫폼 | PowerApps Canvas App |
| 수식 언어 | Power Fx |
| 배포 | Microsoft Teams 탭 앱 |
| 데이터 소스 | SharePoint List (전부) |
| 자동화 | 표준 커넥터만 (Office365Outlook, OneDrive for Business) |
| PDF 생성 | Power Automate 흐름 (OneDrive HTML→PDF 변환, 표준 커넥터) |
| 라이선스 | Power Platform 무료 플랜 |

> **제약**: Dataverse·프리미엄 커넥터·유료 기능을 사용하지 않습니다. 모든 기능은 무료 플랜 범위 내에서 구현되어 있습니다.

---

## 데이터 소스 (SharePoint List)

| List | 역할 |
|--|--|
| `Analysis_Request_DB` | 해석 의뢰 메인 데이터 (현재 상태) |
| `Analysis_Request_Revision_DB` | 수정 이력 + Revision별 첨부파일 |
| `Analysis_Request_Log_DB` | 처리 로그 (등록/승인/반려/배정/완료/담당자변경/제목변경) |
| `Analysis_Report_DB` | 해석 보고서 전용 (의뢰 첨부와 분리) |
| `UserInfo_DB` | 사용자 정보 (이름·이메일·부문·팀·직급) |

---

## 사용자 역할

| 역할 | 권한 |
|--|--|
| 일반 의뢰자 | 의뢰 등록, 본인 의뢰 수정·삭제 |
| 팀장 / 소장 / 이사 | 팀 의뢰 승인·반려 (본인 의뢰도 결재 가능) |
| 가상검증(VA) 승인자 | 담당자 배정 + 승인·반려 (이메일 2명 하드코딩) |
| 가상검증팀 전원 | 전체 진행 내역 조회, 제목 수정, 완료 보고서 수정 |
| 담당자 | 진행 정보 저장, 보고서 업로드, 작업 완료 처리 |

> **조회 권한과 처리 권한 분리**: 가상검증팀 전원은 모든 건을 조회할 수 있지만, 담당자 배정·승인·반려는 VA 승인자 2명(박윤주 팀장, 김동현 책임)만 가능합니다.

---

## 상태 흐름

    PendingTeamLeader  →  PendingVAApproval  →  InProgress  →  Completed
       (팀장 승인대기)       (가상검증 승인대기)      (진행중)        (완료)
                                                        ↘  Rejected (반려)

각 단계 전환 시 관련자에게 이메일이 자동 발송됩니다 (Office365Outlook 표준 커넥터).

---

## 주요 기능

- **칸반 현황판**: 시작전 / 진행중 / 완료 3열 갤러리, 역할별 필터링
- **조회 필터**: 기간(연도)·검색어·완료/반려 포함 토글
- **의뢰 등록·수정**: 폼 기반 입력, 수정 시 사유 필수
- **수정 이력 (Revision)**: 수정할 때마다 스냅샷 + 첨부파일을 Revision별로 분리 저장
- **결재 워크플로우**: 팀장 승인 → 가상검증 승인 + 담당자 배정
- **진행 중 담당자 변경**: 박윤주 팀장이 InProgress 건의 담당자를 상시 재지정 (신규 담당자·의뢰자에게 알림)
- **제목 인라인 수정**: 가상검증팀 전원이 모든 상태에서 해석명을 직접 수정 (연필 아이콘 토글)
- **가상검증팀 전체 조회**: 가상검증팀 전원이 시작전·진행중·완료 전체를 열람
- **진행 관리**: 진행률·예상완료일·작업메모 저장, 의뢰자 이메일 알림
- **보고서 관리**: 완료 시 보고서 업로드 (별도 List), 완료 후에도 가상검증팀이 수정 가능
- **해석 의뢰서 PDF 출력**: Power Automate 흐름으로 HTML→PDF 변환 후 OneDrive 링크 열람 (결재란 포함, 한 페이지)
- **이메일 알림**: 등록·승인·반려·배정·진행·완료·담당자변경 전 단계 자동 발송, 완료 시 가상검증팀 전원 알림

---

## 의뢰번호 체계

    CAE26039
    │  │ └─ 순번 3자리 (해당 연도 누적)
    │  └─── 연도 2자리 (2026 → 26)
    └────── 고정 접두사 (가상검증팀 기존 코드 체계)

- 올해 기존 건수를 반영하여 `CAE26039`부터 시작합니다.
- 연도가 바뀌면 자동으로 `CAE27001`부터 다시 시작됩니다.

---

## 해석 의뢰서 PDF 출력 (Power Automate 방식)

Canvas App의 `Print()` 함수는 화면 뷰포트 크기에 종속되어 긴 문서의 하단이 잘리는 구조적 한계가 있습니다. 이를 해결하기 위해 PDF 생성을 Power Automate 흐름으로 분리했습니다.

**흐름 이름**: `해석보고서_PDF생성`

    [앱] 해석 보고서 출력 버튼 (11개 값 전달)
       ↓
    1. Trigger: When Power Apps calls a flow (V2)
    2. OneDrive Create file: 값을 채운 HTML 파일 생성 (report.html)
    3. OneDrive Convert file: HTML → PDF 변환 (Target type: PDF)
    4. OneDrive Create file: {RequestNo}.pdf 로 저장
    5. OneDrive Create share link: 조직 내 공유 링크 생성 (scope: Organization)
    6. Respond to a PowerApp: PdfLink 반환
       ↓
    [앱] Launch(varPdfResult.pdflink) 로 PDF 열람

- Word Online 커넥터의 PDF 변환은 프리미엄이라, 표준(무료)인 OneDrive `Convert file`을 사용합니다.
- 공유 링크는 테넌트 정책상 Anonymous가 차단되어 있어 `Organization` scope를 사용합니다.
- 흐름은 반드시 **솔루션(Solution)에 포함**되어야 Teams 기반 PowerApps Studio에서 인식됩니다.

---

## 첨부파일 구조 (Revision별 분리)

수정할 때마다 첨부파일이 해당 Revision에 독립적으로 저장됩니다.

    Rev.1  →  1.png          (최초 등록)
    Rev.2  →  2.png          (1차 수정, 새로 올린 파일)
    Rev.3  →  (첨부 없음)     (2차 수정, 텍스트만 변경)

- 각 Revision은 자기만의 첨부파일 세트를 가집니다.
- **"새로 올리기" 방식**: 수정 시 이전 파일이 자동으로 넘어오지 않습니다. 유지하려면 다시 업로드해야 합니다.
- 이는 SharePoint List 간 첨부파일 복사가 PowerApps에서 지원되지 않기 때문에 채택한 구조입니다.
- 수정 화면에서 이전 버전 첨부파일을 읽기 전용으로 확인·다운로드할 수 있습니다.

---

## 화면 구성

- `WorkRequest_Screen` : 메인 화면 (칸반 + 필터 + 등록/조회/수정 팝업 + 결재 + 진행 처리 + 보고서 관리)

상세 화면(View) 팝업은 좌측(요약·상세·역할별 처리 영역) + 우측(처리 현황 타임라인·진행 정보·수정 이력)으로 구성됩니다.

> 기존 `PrintScreen`은 Power Automate PDF 방식으로 전환하면서 삭제되었습니다.

---

## 폴더 구조

    analysis-request-system/
    ├─ README.md                     ← 이 문서 (사람용)
    ├─ CLAUDE.md                     ← Claude 세션 컨텍스트 문서
    ├─ PowerApps/
    │  └─ WorkRequest_Screen.yaml    ← 메인 화면 코드
    ├─ PowerAutomate/
    │  └─ 해석보고서_PDF생성.md       ← PDF 생성 흐름 정의 및 HTML 템플릿
    └─ sharepoint/
       └─ sharepoint_schema.md       ← SharePoint List 컬럼 정의

---

## 개발 환경

혼자 개발·테스트·배포하는 1인 프로젝트입니다. PowerApps Studio에서 YAML 코드를 직접 편집하며, 변경사항은 GitHub로 버전 관리합니다.
