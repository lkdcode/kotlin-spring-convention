# infra — 인프라 서비스

`domains/`, `framework/`, `cloud/` 와 형제 레벨.
자체 운용 가능한 인프라 서버 및 외부 SaaS 연동.

## 구조

```
infra/
├── redis/      ← Cache, Pub/Sub, 분산 락
├── kafka/      ← 메시지 스트리밍
└── slack/      ← 알림 연동
```

## cloud/ 와의 차이

| | `cloud/` | `infra/` |
|---|---|---|
| 예시 | AWS, NCP | Redis, Kafka, Slack |
| 실행 주체 | 클라우드 벤더 전용 | 자체 운용 가능 (Docker 등) |
| 클라이언트 | 벤더 SDK | 범용 클라이언트 라이브러리 |

## 원칙

- `domains/` 는 직접 의존 금지 — `framework/` 포트 인터페이스를 통해 간접 사용

```
domains/ → framework/ → infra/
```
