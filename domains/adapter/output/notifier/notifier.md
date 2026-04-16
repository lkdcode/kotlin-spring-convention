# adapter/output/notifier — 알림 어댑터

특정 수신자(디바이스/사용자)에게 알림을 발송하는 어댑터.
포트: `application/ports/output/notifier/`

## 대상 서비스

| 서비스 | 용도 | 어댑터 위치 |
|---|---|---|
| FCM | 모바일 푸시 알림 | `adapter/output/notifier/fcm/*NotifierAdapter.kt` |
| APNS | iOS 푸시 알림 | `adapter/output/notifier/apns/*NotifierAdapter.kt` |
| Slack | 팀 채널 알림 | `adapter/output/notifier/slack/*NotifierAdapter.kt` |
| SMS | 문자 메시지 | `adapter/output/notifier/sms/*NotifierAdapter.kt` |

```
adapter/output/notifier/
├── notifier.md
├── fcm/
│   └── *NotifierAdapter.kt
├── slack/
│   └── *NotifierAdapter.kt
└── sms/
    └── *NotifierAdapter.kt
```

## 규칙

- 포트 인터페이스 기반 — `application/ports/output/notifier/*Notifier`
- 비즈니스 로직 금지 — 메시지 포맷팅 + 발송만
- 외부 서비스 설정은 `cloud/` 또는 `infra/` 참조
- 어노테이션: `@Service`
- 상세 구현: `fcm/fcm.md` 참조
