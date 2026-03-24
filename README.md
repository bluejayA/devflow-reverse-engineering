# devflow-reverse-engineering

Brownfield 코드베이스의 구조, 흐름, 패턴을 분석하여 문서화하는 Claude Code 플러그인.

## 설치

```bash
claude plugin add /path/to/devflow-reverse-engineering
```

## 사용

```
"이 코드베이스 분석해줘"
"src/auth 모듈의 흐름을 추적해줘"
```

## 실행 모드

- **Lite**: 전체 구조 요약 (INCEPTION 자동 제안용)
- **Deep**: 특정 서브시스템/흐름 심층 분석
- **Delta**: 기존 산출물 + git diff 기반 증분 분석

## 산출물

```
reverse-engineering/
├── README.md       -- 요약, Entry Point Map, 리스크, open questions
├── diagram.mermaid -- 모듈 관계 + 주요 흐름 시각화
└── evidence.md     -- 근거 파일:라인, confidence 태그, 미탐색 범위
```

## aidlc-devflow 연동

이 플러그인은 독립적으로 사용할 수 있습니다. aidlc-devflow가 설치된 환경에서는 INCEPTION 시 Brownfield 감지 후 자동 제안됩니다.

## License

MIT
