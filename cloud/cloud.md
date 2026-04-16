# cloud — 클라우드 벤더

`domains/`, `framework/`, `infra/` 와 형제 레벨.
벤더 종속 클라우드 서비스 SDK.

## 구조

```
cloud/
├── aws/        ← Amazon Web Services
└── ncp/        ← Naver Cloud Platform
```

## 원칙

- 벤더 SDK 직접 사용 — 클라우드에서만 실행되는 서비스
- `domains/` 는 직접 의존 금지 — `framework/` 포트 인터페이스를 통해 간접 사용
- 자체 운용 가능한 인프라 → `infra/`

```
domains/ → framework/ → cloud/
```
