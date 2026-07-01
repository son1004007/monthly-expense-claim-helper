# 전자결재 개인경비 자동화 설계서

## 1. 목적

월초청구서 개인경비 지출결의서 작성 시 반복되는 입력 작업을 줄이기 위해, 사용자가 지정 폴더에 수동으로 넣은 영수증 이미지와 대중교통 이용내역을 읽어 전자결재 작성 화면에 필요한 값만 자동 입력한다.

자동화의 기본 원칙은 다음과 같다.

- 원본 영수증, 교통사용 내역, 계정 정보는 Git 저장소에 저장하지 않는다.
- Python 프로그램 또는 Codex 브라우저 제어가 전자결재 화면에 값을 입력한다.
- 결재요청 제출은 자동화하지 않고, 사용자가 화면에서 최종 확인 후 직접 수행한다.
- 임시저장, 파일첨부, 결재요청처럼 외부 상태를 바꾸는 동작은 실행 직전 사용자 확인을 받는다.

## 2. 범위

### 포함

- 월별 입력 폴더 스캔
- 교통사용 내역 파일 파싱
- 야근식대 텍스트 또는 수기 입력 파일 파싱
- 영수증 이미지 목록 수집
- 엑셀 취소선 기준 사용자 수동 제외 항목 분리
- 집/본사 정류장 기준 정기 출퇴근 구간 제외
- 근태 메모를 이용한 외근/연차/반차/야근 퇴근시간 보강
- 경비 항목 정규화
- 프로젝트 코드와 계정과목 검증
- 전자결재 화면 입력값 생성
- 전자결재 신규 지출결의서 화면 자동 입력
- 첨부파일 업로드 전 확인 및 업로드 보조
- 미리보기 Markdown/JSON 생성

### 제외

- 그룹웨어 로그인 자동화
- 결재요청 자동 제출
- 결재선 임의 변경
- 원본 영수증 이미지 Git 저장
- 계정 비밀번호, 세션 쿠키, 인증 토큰 저장
- 사내 규정상 허용되지 않은 우회 접속 또는 보안 우회

## 3. 현재 기준 입력 자료

사용자는 월별 자료를 아래와 같은 사내 NAS 폴더에 수동으로 넣는다.

```text
D:\02_nas\onycom\01_월초청구서\YYYY년M월 월초청구서
```

현재 확인된 파일 구성 예시는 다음과 같다.

```text
01_근태확인.txt
교통사용_내역.xlsx
교통사용_내역.xls
야근 식대 {일}일.jpg
외근 식대 {일}일.jpg
교통사용_내역_택시{번호}.jpg
```

주의 사항:

- `.~lock.교통사용_내역.xls#` 같은 잠금 파일은 입력 대상에서 제외한다.
- 원본 폴더는 NAS 또는 로컬 전용 위치로 유지한다.
- 이 저장소에는 원본 자료를 복사하지 않는다.

## 4. 목표 전자결재 화면

대상 화면은 ONY Office 전자결재의 `지출결의서(개인경비)` 신규 작성 화면이다.

예시 URL:

```text
https://office.onycom.com/app/approval/document/new/32/939
```

기존 지출결의서 문서 기준으로 확인된 입력 항목은 다음과 같다.

| 구분 | 내용 |
| --- | --- |
| 문서 유형 | 지출결의서(개인경비) |
| 제목 | 지출결의서(개인경비) |
| 결의자 | 기안자 |
| 소속 | 기술개발팀 |
| 직위 | 차장 |
| 결재선 | 기안자 → 1차 결재자 → 2차 결재자 → 회계 담당자 → 최종 결재자 |
| 상세 행 | 지출일, 대상, PJT 약칭, PJT 코드, 계정구분, 계정과목, 계정항목, 금액, 상세내역 |
| 첨부 | 교통사용 내역 파일, 영수증 이미지 |

## 5. 전자결재 필드 매핑

현재 전자결재 신규 작성 화면에서 확인한 DOM 기준 필드 매핑 초안이다. 실제 구현 시 화면 변경 가능성을 고려해 자동화 실행 전 필드 검증을 먼저 수행한다.

| 업무 필드 | 전자결재 필드 후보 | 예시 값 | 비고 |
| --- | --- | --- | --- |
| 제목 | `#subject` | `지출결의서(개인경비)` | 기본값 유지 |
| 지출일 | `#dynamic_table1_{row}_1` | `2026-06-25(목)` | 행별 입력 |
| 대상 | `#dynamic_table1_{row}_2` | `기안자` | 기본값 또는 입력값 |
| PJT 약칭 | `#dynamic_table1_{row}_3` | `인천공항_데이터 분석서비스` | 프로젝트 코드 매핑 필요 |
| PJT 코드 | `#dynamic_table1_{row}_4` | `BDBXX-XXXXX` | 프로젝트 코드 매핑 필요 |
| 계정구분 | `#dynamic_table1_{row}_9` | `판관비` | select |
| 계정과목 | `#dynamic_table1_{row}_5` | `여비교통비` | 경비 유형별 결정 |
| 계정항목 | `#dynamic_table1_{row}_6` | `업무교통비` | 경비 유형별 결정 |
| 금액 | `#dynamic_table1_{row}_7` | `4,450` | 숫자/쉼표 처리 확인 |
| 상세내역 | `#dynamic_table1_{row}_8` | `출발정류장-도착정류장` | 교통 구간 또는 결제 시각 |
| 첨부파일 | `input[name="file"]` | 원본 파일 목록 | 업로드 전 사용자 확인 |

상세 행이 부족하면 화면의 `추가` 버튼을 눌러 행을 생성한 뒤 입력한다. 행 추가 버튼의 DOM은 구현 시 다시 확인한다.

## 6. 데이터 모델

자동화 내부에서는 모든 경비 항목을 아래 모델로 정규화한다.

```json
{
  "claim_month": "2026-07",
  "expense_date": "2026-06-25",
  "target": "기안자",
  "project_alias": "프로젝트A",
  "project_code": "BDBXX-XXXXX",
  "account_group": "판관비",
  "account_subject": "여비교통비",
  "account_item": "업무교통비",
  "amount": 4450,
  "detail": "출발정류장-도착정류장",
  "source_type": "transport",
  "attendance_type": "offsite",
  "work_location": "고객사A",
  "review_status": "claimable",
  "review_reason": "",
  "source_file": "교통사용_내역.xls",
  "attachment_files": ["교통사용_내역.xls"]
}
```

야근식대 항목 예시는 다음과 같다.

```json
{
  "claim_month": "2026-07",
  "expense_date": "2026-06-25",
  "target": "기안자",
  "project_alias": "프로젝트B",
  "project_code": "BDBXX-YYYYY",
  "account_group": "판관비",
  "account_subject": "복리후생비",
  "account_item": "야근식대",
  "amount": 7800,
  "detail": "20:30",
  "source_type": "overtime_meal",
  "attendance_type": "overtime",
  "leave_work_time": "20:30",
  "review_status": "claimable",
  "review_reason": "",
  "source_file": "야근식대.txt",
  "attachment_files": ["receipt-001.jpg"]
}
```

엑셀 취소선으로 사용자가 직접 제외한 항목은 아래처럼 보관하되 전자결재 입력 대상에서는 제외한다.

```json
{
  "claim_month": "2026-07",
  "expense_date": "2026-06-05",
  "target": "기안자",
  "account_group": "판관비",
  "account_subject": "여비교통비",
  "account_item": "업무교통비",
  "amount": 1500,
  "detail": "출발정류장-도착정류장",
  "source_type": "transport",
  "review_status": "manual_excluded_strikethrough",
  "review_reason": "사용자 취소선 제외"
}
```

## 7. 청구 제외 및 계정과목 규칙

### 7.1 취소선 수동 제외

교통사용 내역 엑셀에서 사용자가 취소선으로 표시한 행은 자동화 판단보다 우선해서 제외한다.

처리 기준:

- 취소선이 있는 행은 `manual_excluded_strikethrough`로 표시한다.
- `manual_excluded_strikethrough` 항목은 전자결재 웹브라우저에 자동 입력하지 않는다.
- 행 전체 취소선을 기본으로 하되, 지출일/승차역/하차역/금액 중 하나라도 취소선이면 행 전체 제외로 본다.
- 취소선 서식을 안정적으로 읽기 위해 `교통사용_내역.xlsx`를 우선 사용한다.
- `.xls`만 있는 경우에는 사용자가 `.xlsx`로 저장하거나 Excel COM 기반 리더를 사용한다.
- 미리보기에는 `사용자 취소선 제외` 사유를 남겨 사용자가 확인할 수 있게 한다.

### 7.2 출퇴근 구간 제외

집과 본사 사이의 정기 출퇴근 구간은 결재 요청 대상에서 제외한다.

예시:

```text
집 정류장 -> 본사 정류장
본사 정류장 -> 집 정류장
```

실제 정류장명은 공개 저장소에 남기지 않고 `config/commute-rules.local.ini` 또는 `local/commute-rules.ini`에 기록한다. 예시 파일은 `config/commute-rules.example.ini`로 제공한다.

처리 기준:

- 교통사용 내역의 승차/하차가 출퇴근 제외 구간과 일치하면 `excluded_commute`로 표시한다.
- `excluded_commute` 항목은 전자결재 웹브라우저에 자동 입력하지 않는다.
- 미리보기에는 제외 사유를 남겨 사용자가 확인할 수 있게 한다.
- 정류장명이 바뀌면 local ini 파일만 수정한다.

### 7.3 주말/개인 이동 검토

주말, 공휴일, 연차, 반차일의 이동은 근태 메모와 결합해서 청구 가능 여부를 판단한다.

- 주말/공휴일이고 근태 메모에 `주말출근`, `휴일근무`, `외근` 등 출근 근거가 있으면 `claimable`로 둔다.
- 주말/공휴일인데 근태 메모에 출근 근거가 없으면 `exception_weekend_no_attendance`로 표시한다.
- 연차/반차일 이동은 `review_required`로 표시한다.
- `exception_weekend_no_attendance`, `review_required`는 자동 입력 대상에서 제외한다.
- 사용자가 미리보기에서 삭제하거나 승인한 항목만 최종 입력 대상으로 전환한다.

### 7.4 근태 메모 활용

초기 규칙은 기존 지출결의서 입력 내역을 기준으로 한다.

| 경비 유형 | 계정구분 | 계정과목 | 계정항목 | 상세내역 |
| --- | --- | --- | --- | --- |
| 대중교통 | 판관비 | 여비교통비 | 업무교통비 | 출발지-도착지 |
| 야근식대 | 판관비 | 복리후생비 | 야근식대 | 결제 시각 또는 사용처 |

사용자가 작성한 근태 메모는 아래 용도로 사용한다.

- 외근일: 교통비 청구 후보가 외근일 이동인지 검증한다.
- 야근일: 야근식대 상세내역에 퇴근 시간을 입력한다.
- 연차/반차일: 교통 이동을 `review_required`로 표시한다.
- 주말/공휴일: 출근 근거가 없는 대중교통 이동을 `exception_weekend_no_attendance`로 표시한다.

사용자 입력 예시는 `config/attendance-notes.example.md`에 둔다. 실제 월별 근태 메모는 `local/attendance/YYYY-MM.md` 또는 `attendance-notes/YYYY-MM.md`에 작성하고 Git에 올리지 않는다.

### 7.5 설정 파일

프로젝트 코드와 약칭은 별도 설정 파일로 관리한다.

예정 설정 파일:

```text
config/project-codes.yml
config/account-rules.yml
config/commute-rules.local.ini
local/attendance/YYYY-MM.md
```

설정 파일에는 실제 내부 프로젝트 정보가 들어갈 수 있으므로 공개 저장소에 올릴 때는 예시 파일만 커밋한다.

```text
config/project-codes.example.yml
config/account-rules.example.yml
config/commute-rules.example.ini
config/attendance-notes.example.md
```

## 8. 파이프라인

### 8.1 입력 스캔

명령 예시:

```powershell
python -m expense_claim scan --source "D:\02_nas\onycom\01_월초청구서\26년7월 월초청구서" --claim-month 2026-07
```

처리 내용:

- 입력 폴더 존재 여부 확인
- `.~lock.*` 잠금 파일 제외
- `교통사용_내역.xls` 존재 여부 확인
- `야근식대.txt` 존재 여부 확인
- 이미지 파일 목록 수집
- 파일명, 크기, 수정일, SHA-256 해시로 manifest 생성

출력 예시:

```text
runs/2026-07/manifest.json
```

### 8.2 원천 파싱

명령 예시:

```powershell
python -m expense_claim extract --manifest runs/2026-07/manifest.json
```

처리 내용:

- 교통사용 내역 `.xls` 파일을 읽어 교통비 후보 행 생성
- 같은 폴더에 `교통사용_내역.xlsx`가 있으면 `.xlsx`를 우선 사용해 취소선 서식을 읽음
- 취소선 행은 `manual_excluded_strikethrough`로 표시하고 입력 대상에서 제외
- `야근식대.txt` 파일을 읽어 야근식대 후보 행 생성
- 이미지 파일은 자동 OCR하지 않고 우선 첨부 후보로만 관리
- 필요한 경우 이미지 파일명과 텍스트 내역을 수동 매핑

출력 예시:

```text
runs/2026-07/raw-items.json
```

### 8.3 정규화 및 검증

명령 예시:

```powershell
python -m expense_claim build --raw runs/2026-07/raw-items.json --project-codes config/project-codes.yml
```

처리 내용:

- 날짜 형식 정규화
- 금액 숫자화
- 출발지/도착지 정규화
- 집/본사 정류장 기준 정기 출퇴근 구간 제외
- 근태 메모 기준 외근/연차/반차/야근 퇴근시간 보강
- 프로젝트 약칭/코드 보강
- 계정구분/계정과목/계정항목 자동 결정
- 필수값 누락 확인
- 합계 금액 계산
- 첨부파일 누락 확인
- 중복 파일 해시 확인
- `claimable`, `manual_excluded_strikethrough`, `excluded_commute`, `review_required`, `exception_weekend_no_attendance` 상태 부여

출력 예시:

```text
runs/2026-07/claim-items.json
runs/2026-07/claim-preview.md
```

### 8.4 사용자 검토

자동 입력 전 사용자가 `claim-preview.md`를 확인한다.

검토 항목:

- 지출일이 맞는지
- 프로젝트 코드가 맞는지
- 계정과목/계정항목이 맞는지
- 취소선으로 표시한 항목이 입력 대상에서 빠졌는지
- 출퇴근 제외 구간이 입력 대상에서 빠졌는지
- 주말/공휴일 이동 중 출근 근거가 없는 항목이 예외 대상으로 분리됐는지
- 연차/반차 이동이 검토 대상으로 분리됐는지
- 외근 교통비 상세내역이 승하차 내용으로 작성됐는지
- 야근식대 상세내역이 퇴근 시간으로 작성됐는지
- 금액 합계가 영수증/교통사용 내역과 맞는지
- 첨부할 파일이 빠지지 않았는지
- 결재 문서 제목과 결재선이 맞는지

### 8.5 브라우저 입력

명령 예시:

```powershell
python -m expense_claim fill `
  --items runs/2026-07/claim-items.json `
  --url "https://office.onycom.com/app/approval/document/new/32/939" `
  --dry-run
```

처리 내용:

- 전자결재 신규 문서 화면이 열려 있는지 확인
- 문서 유형이 `지출결의서(개인경비)`인지 확인
- 제목 기본값 확인
- 필요한 행 수 확인 후 부족하면 행 추가
- `claimable` 상태의 상세 행만 입력
- `manual_excluded_strikethrough`, `excluded_commute`, `review_required`, `exception_weekend_no_attendance` 상태의 행은 입력하지 않고 미리보기에서만 표시
- 합계가 화면 합계와 일치하는지 확인
- 첨부파일 업로드 전 사용자 확인
- 결재요청은 수행하지 않고 입력 완료 상태에서 멈춤

`--dry-run` 모드에서는 실제 입력하지 않고 입력 대상 필드와 값을 출력한다.

### 8.6 최종 확인

사용자는 브라우저에서 다음을 확인한다.

- 전체 합계
- 상세 행 내용
- 첨부파일 목록
- 결재선
- 공개/비공개, 보존연한 등 문서정보

확인 후 사용자가 직접 `결재요청`을 클릭한다.

## 9. 구현 방식

### 9.1 Codex 브라우저 제어 방식

초기 구현은 Codex가 사용자의 Chrome에 열린 ONY Office 세션을 제어하는 방식이 가장 단순하다.

장점:

- 사용자가 이미 로그인한 Chrome 세션을 활용할 수 있다.
- 별도 로그인 자동화가 필요 없다.
- 현재 화면의 DOM을 확인하면서 필드 매핑을 조정할 수 있다.

제약:

- Codex 실행 환경과 Chrome 확장 연결이 필요하다.
- 자동화 로직을 일반 CLI 도구로 재사용하기 어렵다.

### 9.2 Python + Playwright 방식

반복 실행 가능한 프로그램으로 만들려면 Python + Playwright 또는 Chrome CDP 연결 방식을 사용한다.

권장 구조:

- 사용자가 Chrome을 로그인 상태로 준비
- 자동화 프로그램이 기존 Chrome 또는 별도 Playwright 브라우저에 연결
- 세션이 없으면 사용자가 직접 로그인한 뒤 다음 단계 진행

주의:

- 로그인 정보 저장은 기본 비활성화한다.
- `storage-state.json`, 쿠키 파일은 `.gitignore`에 포함하고 Git에 올리지 않는다.
- 결재요청 같은 제출 동작은 기본 비활성화한다.

## 10. 예정 디렉터리 구조

```text
monthly-expense-claim-helper/
  README.md
  docs/
    e-approval-expense-automation-design.md
    input-format-and-review-rules.md
  config/
    account-rules.example.yml
    attendance-notes.example.md
    commute-rules.example.ini
    project-codes.example.yml
  schemas/
    claim-item.schema.json
  src/
    expense_claim/
      __init__.py
      scan.py
      extract.py
      normalize.py
      validate.py
      preview.py
      browser_fill.py
  tests/
    fixtures/
      README.md
  runs/                  # ignored
  local/                 # ignored
  attendance-notes/      # ignored
```

## 11. 검증 기준

### 1차 완료 기준

- 지정 폴더에서 원천 파일 목록을 정확히 수집한다.
- 잠금 파일과 임시 파일을 제외한다.
- 경비 항목 JSON과 Markdown 미리보기를 생성한다.
- 엑셀 취소선 행을 전자결재 입력 대상에서 제외한다.
- 출퇴근 제외 구간을 전자결재 입력 대상에서 제외한다.
- 근태 메모를 파싱해 야근식대 퇴근 시간과 외근일 검증 정보를 보강한다.
- 전자결재 입력 전 누락/오류를 사용자에게 보여준다.

### 2차 완료 기준

- 전자결재 신규 문서 화면의 필드 구조를 검증한다.
- `claimable` 경비 항목만 화면 상세 행에 자동 입력한다.
- 화면 합계와 내부 합계가 일치하는지 확인한다.
- 입력 후 결재요청 전 상태에서 멈춘다.

### 3차 완료 기준

- 첨부파일 업로드를 사용자 확인 후 수행한다.
- 입력 완료 후 검토 체크리스트를 표시한다.
- 월별 반복 실행 시 중복 입력 위험을 감지한다.

## 12. 실패 처리

| 상황 | 처리 |
| --- | --- |
| 입력 폴더 없음 | 즉시 중단하고 경로 안내 |
| `.xls` 파일 잠금 감지 | 읽기 가능 여부 확인, 필요 시 사용자가 엑셀 종료 |
| 프로젝트 코드 미확인 | 해당 행을 오류로 표시하고 자동 입력 제외 |
| 금액 파싱 실패 | 원문과 함께 오류 표시 |
| 첨부 이미지 부족 | 경고 표시, 사용자 확인 전 입력 중단 |
| 엑셀 취소선 행 | `manual_excluded_strikethrough`로 표시하고 자동 입력 제외 |
| 출퇴근 제외 구간 | `excluded_commute`로 표시하고 자동 입력 제외 |
| 주말/공휴일인데 출근 근거가 없는 이동 | `exception_weekend_no_attendance`로 표시하고 자동 입력 제외 |
| 연차/반차일 이동 | `review_required`로 표시하고 자동 입력 제외 |
| 야근식대 퇴근시간 없음 | `review_required`로 표시하고 자동 입력 제외 |
| 전자결재 화면 아님 | 자동 입력 중단 |
| 필드 ID 변경 | DOM 스냅샷 저장 후 매핑 갱신 필요 |
| 합계 불일치 | 자동 입력 중단 |

## 13. 보안 및 감사 원칙

- 원본 파일은 NAS 또는 로컬 폴더에만 둔다.
- Git에는 설계서, 예시 설정, 스크립트만 올린다.
- 실행 결과도 기본적으로 `runs/`에 저장하고 Git에서 제외한다.
- 집/본사 정류장명과 실제 근태 메모는 개인정보에 가까운 자료이므로 local 설정으로 관리하고 Git에 올리지 않는다.
- 파일 해시는 중복 확인과 감사용으로만 사용한다.
- 브라우저 자동화는 입력과 검증까지만 수행한다.
- 결재요청은 사용자 최종 확인 후 직접 수행한다.

## 14. 다음 작업

1. `project-codes.example.yml`과 `account-rules.example.yml` 예시 작성
2. `claim-item.schema.json` 작성
3. `commute-rules.local.ini` 로더 구현
4. 근태 메모 파서 구현
5. 입력 폴더 스캐너 구현
6. 교통사용 내역 `.xls` 파서 구현
7. `야근식대.txt` 파서 구현
8. 엑셀 취소선, 출퇴근 제외, 주말 출근 근거, 검토 필요 상태 분류 구현
9. `claim-preview.md` 생성기 구현
10. 전자결재 화면 필드 검증 스크립트 구현
11. dry-run 입력 계획 출력 구현
12. Chrome/Playwright 자동 입력 구현
13. 첨부파일 업로드 확인 절차 구현
