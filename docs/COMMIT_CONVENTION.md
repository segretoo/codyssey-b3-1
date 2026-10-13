# 커밋 / PR 컨벤션

## 기존 이력 분석 (git log --oneline -15)
- 대부분 `docs: 한글 요약` 형식이다. (예: `docs: IAM 사용자 로그인 증빙 추가, 정리 체크리스트 보완`)
- 예외 1: `docs:change` — 공백이 없고 무엇을 바꿨는지 알 수 없다.
- 예외 2: `Update README.md for consistency in formatting` — GitHub 웹 편집기 기본 문구가 그대로 남았다.
- 결론: 접두어는 유지하고, 공백과 구체적인 한글 요약을 규칙으로 명시한다.

## 커밋 메시지
- 형식: `type(scope): 한글 요약` — scope를 반드시 쓴다. (예: `docs(readme)`, `docs(evidence)`)
- type: `feat`(기능), `fix`(버그), `docs`(문서), `refactor`(구조 개선), `chore`(설정/기타)
- 제목은 50자 이내, 끝에 마침표를 쓰지 않고, `type:` 뒤에 공백 한 칸을 둔다.
- 본문은 불릿으로 "왜"를 쓰고, 파일명은 불릿 안에서만 언급한다. 파일명만 나열하는 줄은 쓰지 않는다.
- GitHub 웹 편집기의 기본 문구(`Update README.md ...`)는 쓰지 않는다.

### 예시
- `docs(readme): 외부 접속 증빙 절차 보강`
- `docs(evidence): IGW·보안 그룹 캡처 추가`

## Pull Request
- 제목에도 type 접두어를 붙인다. (예: `docs: PR 템플릿 및 커밋 컨벤션 추가`)
- 본문은 PR 템플릿의 Why / What / How to Test를 채운다.
- 민감정보(키, 비밀번호, 필요 이상의 IP·계정 정보)는 올리지 않는다.
