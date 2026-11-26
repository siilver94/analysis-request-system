# 구성 가이드

이 저장소의 코드로 같은 앱을 직접 구성하는 방법입니다.
Microsoft 365 조직 계정과 Teams, SharePoint 사이트를 만들 수 있는 권한이 필요합니다. 추가 라이선스는 필요하지 않습니다.

## 목차

1. [준비물](#1-준비물)
2. [SharePoint List 만들기](#2-sharepoint-list-만들기)
3. [앱 만들기](#3-앱-만들기)
4. [예시 값 바꾸기](#4-예시-값-바꾸기)
5. [PDF 흐름 만들기](#5-pdf-흐름-만들기)
6. [권한과 공유](#6-권한과-공유)
7. [역할별 테스트](#7-역할별-테스트)
8. [게시 전 점검](#8-게시-전-점검)
9. [자주 겪는 문제](#9-자주-겪는-문제)

---

## 1. 준비물

| 항목 | 용도 |
|--|--|
| Microsoft Teams 팀 | 앱을 탭으로 추가할 곳 |
| SharePoint 사이트 | List를 둘 곳 (Teams 팀에 연결된 사이트도 가능) |
| Power Apps (Teams용) | 앱 편집 |
| Power Automate | PDF 흐름 |
| OneDrive for Business | PDF 변환과 저장 |

---

## 2. SharePoint List 만들기

[sharepoint/sharepoint_schema.md](../sharepoint/sharepoint_schema.md)의 정의대로 List를 만듭니다.

| List | 직접 만들기 | 비고 |
|--|--|--|
| `Analysis_Request_DB` | 필요 | 의뢰 본문 |
| `Analysis_Request_Revision_DB` | 필요 | 사용자 컬럼은 모두 **필수 해제** |
| `Analysis_Request_Log_DB` | 필요 | `LogType` 선택지 11개 등록 |
| `Analysis_Report_DB` | 필요 | 문서 라이브러리가 아닌 **일반 List** |
| `UserInfo_DB` | 필요 | 사용자별로 이름, 이메일, 부문, 팀, 직급 입력 |
| `Project_DB` | 선택 | 조직에 프로젝트 목록이 없다면 `업무_계정`, `프로젝트명` 두 컬럼만 가진 List로 대체 |

만들 때 주의할 점입니다.

- **List 이름과 컬럼 이름은 문서와 같게** 만듭니다. 이름이 다르면 수식이 모두 오류가 납니다.
- **컬럼은 처음 만들 때의 이름이 내부 이름**이 됩니다. 영문으로 만든 뒤 표시 이름만 바꾸면 수식에는 영향이 없습니다.
- `Status`와 `LogType`은 **선택(Choice)** 형식이며, 선택지 값은 문서의 영문 값과 철자가 같아야 합니다.
- `UserInfo_DB`의 이메일은 소문자로 넣는 것을 권장합니다. 앱은 소문자로 바꿔 비교합니다.
- `UserInfo_DB`에 **팀별 팀장이 한 명씩** 있어야 합니다. 등록 시 요청자와 같은 팀에서 직급이 "팀장"인 사람을 찾아 승인자로 지정합니다.

---

## 3. 앱 만들기

1. Teams에서 Power Apps 앱을 열고 팀을 선택해 새 앱을 만듭니다.
2. **데이터 추가**에서 위 List 6개와 **Office 365 Outlook**을 연결합니다.
3. 기본 화면의 이름을 `WorkRequest_Screen`으로 바꿉니다.
4. `PowerApps/WorkRequest_Screen.yaml`의 내용을 화면에 붙여 넣습니다. 트리 뷰에서 화면을 선택하고 **코드 붙여넣기**를 사용합니다(Studio 버전에 따라 메뉴 이름이 다를 수 있습니다).
5. 수식 오류가 남아 있으면 대부분 데이터 원본 이름이나 컬럼 이름이 다른 경우입니다. 오류가 난 수식의 이름을 List와 비교합니다.

> YAML의 `Control: Button@0.0.45` 같은 표기는 컨트롤 버전입니다. Studio 버전에 따라 일부 최신 컨트롤이 붙여 넣어지지 않으면, 같은 종류의 컨트롤을 직접 추가한 뒤 속성만 옮깁니다.

---

## 4. 예시 값 바꾸기

공개용 코드에는 실제 주소 대신 예시 값이 들어 있습니다. 아래 항목을 사용하는 조직에 맞게 바꿉니다.

| 위치 | 예시 값 | 바꿀 내용 |
|--|--|--|
| `WorkRequest_Screen.OnVisible` › `varIsVAApprover` | `va.lead@example.com`, `va.approver@example.com` | 해석팀 승인자 이메일 |
| `WorkRequest_Screen.OnVisible` › `varVAApprover1Email`, `varVAApprover2Email` | 위와 같음 | 승인 요청 메일을 받을 주소 |
| `WorkRequest_Screen.OnVisible` › `varIsAssigneeManager` | `va.lead@example.com` | 진행 중 담당자를 바꿀 수 있는 사람 |
| `WorkRequest_Screen.OnVisible` › `varIsAdmin` | `app.admin@example.com`, `shared.admin@example.com` | 전체 건을 볼 관리자 |
| `WorkRequest_Screen.OnVisible` › `varIsVATeamMember` | `"가상검증팀"` | 해석을 수행하는 팀의 이름 (`UserInfo_DB`의 팀 값) |
| `drpAssignee.Items`, `btnCompleteAnalysis.OnSelect` | `"가상검증팀"` | 위와 같은 팀 이름 |
| `WorkRequest_Screen.OnVisible` › `colWorkCategoryOptions` | 제품개발, 기술개발, 설계변경, 원가절감 | 업무 계정 분류 |
| `frmAnalysisRequest.OnSuccess` › `varNewRequestNo` | 접두사 `CAE`, 오프셋 | 의뢰번호 접두사와 시작 번호 |
| `PowerAutomate/report_template.html` | `CENTRAL R&D CENTER`, `기술연구소` | 문서에 표시할 조직 이름 |

팀 이름은 여러 수식에 문자열로 들어 있습니다. Studio의 검색 기능으로 `가상검증팀`을 찾아 한 번에 확인하는 것이 안전합니다.

---

## 5. PDF 흐름 만들기

[PowerAutomate/README.md](../PowerAutomate/README.md)의 순서대로 흐름을 만듭니다. 순서를 요약하면 다음과 같습니다.

1. Power Automate에서 **솔루션**을 만들고 그 안에서 인스턴트 클라우드 흐름을 만듭니다. 트리거는 **Power Apps (V2)** 입니다.
2. 텍스트 입력 11개를 문서의 순서대로 추가합니다.
3. OneDrive for Business 액션 4개(Create file, Convert file, Create file, Create share link)와 응답 액션을 추가합니다.
4. 흐름 이름을 `해석보고서_PDF생성`으로 저장합니다.
5. 앱의 Power Automate 패널에서 **흐름 추가**로 이 흐름을 연결합니다.

흐름 이름을 다르게 지으면 `btnPrintReport.OnSelect`의 호출 이름도 같이 바꿉니다.

---

## 6. 권한과 공유

- 앱 사용자는 List에 **편집 권한**이 있어야 합니다. 앱이 사용자 본인의 권한으로 List를 읽고 씁니다.
- `UserInfo_DB`와 `Project_DB`는 읽기 권한만으로 충분합니다.
- 흐름을 사용할 사람에게 흐름 실행 권한이 있어야 합니다. 앱에 흐름을 추가하고 게시하면 사용자가 처음 실행할 때 연결 동의를 묻습니다.
- 앱은 화면에서 역할별로 보이는 범위를 나눌 뿐입니다. List를 직접 열면 모든 행이 보이므로, List 주소를 공유하지 않거나 List 수준 권한을 따로 설계합니다.

---

## 7. 역할별 테스트

혼자 테스트한다면 `UserInfo_DB`에서 본인의 팀과 직급을 바꿔 가며 역할을 전환할 수 있습니다. 승인자와 관리자는 `OnVisible`의 이메일을 잠시 본인 주소로 바꿔 확인합니다.

### 요청자

- [ ] 필수 항목을 비우면 등록되지 않고 안내가 나온다
- [ ] 등록하면 "시작전" 열에 "팀장 승인대기"로 나타난다
- [ ] 의뢰번호가 형식대로 부여된다
- [ ] 팀장에게 승인 요청 메일이 간다
- [ ] 수정할 때 수정 사유를 비우면 저장되지 않는다
- [ ] 수정하면 수정 이력에 새 Revision이 생기고 "최신" 표시가 옮겨 간다
- [ ] 팀장 승인 전에만 삭제 버튼이 보인다
- [ ] 다른 사람의 의뢰는 목록에 보이지 않는다

### 팀장

- [ ] 팀원의 승인대기 건이 "시작전" 열에 보인다
- [ ] 승인하면 "가상검증 승인대기"로 바뀌고 해석팀 승인자와 요청자에게 메일이 간다
- [ ] 반려 사유를 비우면 반려되지 않는다
- [ ] 반려하면 "완료" 열에 "반려"로 나타나고 요청자에게 메일이 간다

### 해석팀 승인자

- [ ] 담당자를 선택하지 않으면 승인 버튼이 비활성화된다
- [ ] 승인하면 "진행중" 열로 옮겨 가고 담당자와 요청자에게 메일이 간다
- [ ] 승인대기 상태에서 "담당자 변경" 버튼이 승인 버튼을 가리지 않는다
- [ ] (해석팀 팀장) 진행중 건에서 담당자를 바꿀 수 있고, 새 담당자와 요청자에게 메일이 간다

### 담당자

- [ ] 본인 담당 건에서만 작업 처리 영역이 보인다
- [ ] 진행률, 예상 완료일, 작업 메모가 저장되고 카드에 반영된다
- [ ] 보고서를 올리기 전에는 "작업 완료"가 비활성화된다
- [ ] 완료하면 요청자, 요청자의 팀장, 해석팀 전원에게 메일이 간다

### 해석팀 일반 구성원

- [ ] 세 열에서 모든 건이 보인다
- [ ] 승인, 반려, 담당자 지정 버튼은 보이지 않는다
- [ ] 연필 아이콘으로 제목을 바꿀 수 있다
- [ ] 완료 건에서 보고서를 교체할 수 있고, 취소하면 원래대로 돌아간다

### 공통

- [ ] 완료 건에서 "해석 보고서 출력"을 누르면 PDF가 새 창으로 열린다
- [ ] PDF의 결재란에 팀장 승인자와 승인일이 나온다
- [ ] 연도 필터, 검색, 완료 · 반려 토글이 동작한다

---

## 8. 게시 전 점검

- [ ] 예시 이메일(`@example.com`)이 코드에 남아 있지 않다
- [ ] `UserInfo_DB`에 사용자 전원과 팀별 팀장이 등록되어 있다
- [ ] `LogType` 선택지 11개가 모두 등록되어 있다
- [ ] 흐름이 솔루션 안에 있고 앱과 같은 환경에 있다
- [ ] 의뢰번호 오프셋이 의도한 시작 번호와 맞다
- [ ] 역할별 테스트를 모두 마쳤다
- [ ] 앱을 게시하고 Teams 탭에서 새 버전이 열리는지 확인했다

---

## 9. 자주 겪는 문제

| 증상 | 확인할 것 |
|--|--|
| 붙여 넣은 뒤 수식이 온통 오류 | 데이터 원본 이름과 List 이름이 같은지, 연결이 추가되었는지 |
| 등록했는데 팀장에게 메일이 안 감 | `UserInfo_DB`에 같은 팀의 직급 "팀장" 행이 있는지, 이메일이 맞는지 |
| 팀장인데 결재 영역이 안 보임 | `UserInfo_DB`에 로그인 이메일과 같은 행이 있고 직급 값이 맞는지 |
| 수정 저장 시 오류 | `Analysis_Request_Revision_DB`의 사용자 컬럼에 필수 설정이 남아 있는지 |
| 로그 저장 시 오류 | `LogType` 선택지에 해당 값이 있는지 |
| 흐름이 앱 목록에 안 보임 | 흐름이 솔루션 안에 있는지, 환경이 같은지 |
| PDF 변환 단계에서 400 오류 | File 입력에 Path가 아닌 Id를 넣었는지 |
| 공유 링크 단계에서 오류 | Link scope가 Organization인지 |
| 목록에 일부 건이 안 보임 | 데이터 행 한도를 넘지 않았는지 ([위임 제약](lessons-learned.md#12-위임되지-않는-필터를-알고-쓴다)) |
