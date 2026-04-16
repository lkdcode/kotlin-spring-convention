# infra/slack — Slack

Webhook 기반 알림. `SlackService` → `SlackMessageCreator` → Slack API.

## 설정

```yaml
slack:
  webhook:
    url: {Incoming Webhook URL}
    alarm-url: {Alarm 채널 Webhook URL}
```

## SlackProperties

```kotlin
@Component
data class SlackProperties(
    @Value("\${slack.webhook.url}") val url: String,
    @Value("\${slack.webhook.alarm-url}") val alarmUrl: String,
)
```

## SlackService

```kotlin
@Service
class SlackService(
    private val slackProperties: SlackProperties,
    private val slackMessageCreator: SlackMessageCreator,
) {
    fun sendErrorAlert(message: String) {
        send(
            payload = slackMessageCreator.errorMessage(message),
            url = slackProperties.alarmUrl,
            logMessage = "슬랙 알림 발송 실패",
        )
    }

    fun sendInfo(message: String) {
        send(
            payload = slackMessageCreator.infoMessage(message),
            url = slackProperties.url,
            logMessage = "슬랙 메시지 발송 실패",
        )
    }

    private fun send(payload: Payload, url: String, logMessage: String) {
        try {
            SLACK.send(url, payload)
        } catch (e: IOException) {
            logWarn(logMessage)
        }
    }

    companion object {
        private val SLACK = Slack.getInstance()
    }
}
```

## SlackMessageCreator

Block Kit 으로 메시지 구성. 프로젝트별 메서드 추가.

```kotlin
@Service
class SlackMessageCreator {

    fun errorMessage(message: String): Payload =
        Payload.builder()
            .blocks(
                Blocks.asBlocks(
                    Blocks.divider(),
                    Blocks.section { section ->
                        section.fields(
                            listOf(
                                BlockCompositions.markdownText("*🔴 오류 발생*"),
                                BlockCompositions.markdownText(message),
                                BlockCompositions.markdownText(TimeUtil.nowKSTTimeFormat()),
                            )
                        )
                    },
                    Blocks.divider(),
                )
            ).build()

    fun infoMessage(message: String): Payload =
        Payload.builder()
            .blocks(
                Blocks.asBlocks(
                    Blocks.section { section ->
                        section.fields(
                            listOf(
                                BlockCompositions.markdownText("*🟢 알림*"),
                                BlockCompositions.markdownText(message),
                                BlockCompositions.markdownText(TimeUtil.nowKSTTimeFormat()),
                            )
                        )
                    }
                )
            ).build()
}
```

## 규칙

- 메시지 구성 로직은 `SlackMessageCreator` 에 — `SlackService` 에 직접 작성 금지
- 발송 실패는 `logWarn` 으로만 처리 — 비즈니스 실패로 간주하지 않음
- Webhook URL 은 설정 파일로 관리 — 하드코딩 금지
- `alarm-url` = 긴급 알림 채널 / `url` = 일반 알림 채널
