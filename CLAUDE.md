# devflow-reverse-engineering 프로젝트 규칙

## GitHub Flow (필수)

### gh CLI 사용
GitHub 작업은 모두 `gh` CLI로 수행한다.

### 커밋 메시지
- Conventional Commits 형식: `feat:`, `fix:`, `chore:`, `docs:`
- HEREDOC으로 멀티라인 메시지 작성

### PR 생성 규칙
- body 첫 줄에 `Closes #N`
- Summary + Test plan 포함
- 테스트 통과 확인 후 PR 생성
- 승인 없이 push/merge 금지
