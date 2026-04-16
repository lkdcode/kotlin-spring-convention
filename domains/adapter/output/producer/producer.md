# adapter/output/producer — 이벤트 발행 어댑터

메시지 브로커(Kafka, RabbitMQ 등)로 이벤트를 발행하는 어댑터.
포트: `application/ports/output/producer/`

## 대상 브로커

| 브로커 | 어댑터 위치 |
|---|---|
| Kafka | `adapter/output/producer/kafka/*ProducerAdapter.kt` |
| RabbitMQ | `adapter/output/producer/rabbitmq/*ProducerAdapter.kt` |

```
adapter/output/producer/
├── producer.md
├── kafka/
│   └── *ProducerAdapter.kt
└── rabbitmq/
    └── *ProducerAdapter.kt
```

## 규칙

- 포트 인터페이스 기반 — `application/ports/output/producer/*Producer`
- 비즈니스 로직 금지 — 메시지 직렬화 + 발행만
- 브로커별 설정은 `infra/kafka/`, `infra/rabbitmq/` 참조
- 어노테이션: `@Service`
- 상세 구현: `kafka/kafka.md` 참조
