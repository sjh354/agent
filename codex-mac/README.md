# Mac Codex 작업 환경

2026-10-30 KST에 이 Mac의 `~/.codex/`에서 복사한 작업 규칙과 설정이다.
기존 `../codex/` 환경과 별도로 보관한다. 원본 규칙·설정 파일의 내용은 변경하지 않았다.

## 포함 범위

- `AGENTS.md`: 전역 작업 지침.
- `rules/`: 공통 상세 규칙과 `serving/` 운영 지침. `legacy-sandbox.md`는 현재 적용하지 않는 과거 참고 자료다.
- `config.toml`: 모델·권한·플러그인·MCP·알림 설정을 포함한 사용자 설정 원본.
- `skills/find-skills/`: 사용자 영역에 설치된 스킬.

## Hook과 MCP

별도 사용자 `hooks/` 또는 `hooks.json`은 발견되지 않았다. `config.toml`의 `notify`는
Mac에 설치된 Computer Use 앱의 `SkyComputerUseClient`를 `turn-ended` 인자로 호출한다.
실행 파일 자체는 포함하지 않았다.

MCP는 `node_repl`과 `loki-local`이 등록되어 있다.
`node_repl`은 설치된 ChatGPT 앱과 플러그인 런타임에 의존하며, `env_vars`에는
상속할 환경 변수 이름만 있다. 해당 환경 변수의 실제 값은 포함하지 않는다.
`loki-local`은 Homebrew의 `npx`로 `simple-loki-mcp@0.0.9`를 실행하고
`http://127.0.0.1:3100`의 Loki에 연결한다. homeserver 조회 시 SSH 터널 준비는
`rules/serving/homeserver.md`와 `operations.md`를 따른다.

## 복원 시 주의

파일을 사용할 때는 대상 Mac의 기존 `~/.codex/AGENTS.md`, `rules/`, `config.toml`을
먼저 백업하고 필요한 파일을 복사하거나 병합한다. `config.toml`의 사용자 절대 경로,
앱 설치 경로·버전, 플러그인 경로와 프로젝트 신뢰 목록은 원본 그대로이므로 대상 환경에서 검토한다.
ChatGPT/Codex 앱, Homebrew·Node.js와 플러그인 설치는 별도로 준비한다.

인증 파일·토큰·비밀번호·세션·로그·캐시·다운로드된 플러그인·내장 스킬·앱 바이너리는
복사하지 않았다. 이 폴더를 저장소에 보관해도 이 Mac의 활성 설정은 변경되지 않는다.

Codex 설정 항목의 의미는 [공식 설정 문서](https://learn.chatgpt.com/docs/config-file/config-reference)를 참고한다.
