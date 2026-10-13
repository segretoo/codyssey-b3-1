# 커밋 / PR 컨벤션

## 커밋 메시지
- 형식: `type(scope): 요약`
- type: `feat`(기능), `fix`(버그), `docs`(문서), `refactor`(구조 개선), `chore`(설정/기타)
- scope는 변경 영역을 짧게 쓴다. (예: `infra`, `readme`, `network`)
- 제목은 50자 이내, 끝에 마침표를 쓰지 않는다.
- 본문은 "무엇을"보다 "왜"가 드러나게 불릿으로 쓴다.

### 예시
- `docs(readme): 외부 접속 증빙 절차 보강`
- `chore(infra): 보안 그룹 설정 메모 추가`

## Pull Request
- 제목은 변경 내용을 한 줄로 요약한다.
- 본문은 PR 템플릿의 Why / What / How to Test를 채운다.
- 민감정보(키, 비밀번호, 필요 이상의 IP·계정 정보)는 올리지 않는다.
