# Monthly Expense Claim Helper

월초청구서 개인경비 지출결의서 작성을 보조하기 위한 자동화 설계 및 구현 저장소입니다.

이 저장소에는 영수증 원본, 교통사용 내역 원본, 그룹웨어 계정 정보, 쿠키, 토큰을 저장하지 않습니다.

## 현재 단계

- 전자결재 자동화 파이프라인 설계 문서 작성
- 입력 자료는 사용자가 사내 NAS의 월별 폴더에 수동으로 넣는 방식을 기본으로 함
- Python만으로 완전 자동화하기보다 Codex가 월별 예외를 판단하고 반복 입력을 보조하는 구조로 운영 예정
- 전자결재 화면 입력은 자동화하되, 결재요청은 사용자가 직접 수행

## 문서

- [전자결재 개인경비 자동화 설계서](docs/e-approval-expense-automation-design.md)
- [입력 포맷 및 검토 규칙](docs/input-format-and-review-rules.md)
- [Codex 보조 월초청구서 처리 운영 절차](docs/codex-assisted-expense-workflow.md)
