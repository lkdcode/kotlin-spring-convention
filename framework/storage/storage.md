# framework/storage — 파일 저장소

## 3가지 저장 방식

| | Direct | Local Pre-signed | S3 Pre-signed |
|---|---|---|---|
| 업로드 흐름 | Client → Server (multipart) | Client → 로컬 URL → Server 확인 | Client → S3 → Server 확인 |
| 서버 파일 처리 | ✅ 직접 처리 | ✅ 직접 처리 | ❌ S3 처리 |
| 서버 부하 | 파일 크기만큼 | 파일 크기만큼 | URL 발급만 |
| 구현 복잡도 | 단순 | 중간 | 중간 |
| 프론트 코드 | 별도 구현 | S3와 동일 | S3와 동일 |
| 로컬 ↔ 운영 교체 | 코드 변경 필요 | 구현체만 교체 | 구현체만 교체 |

## 선택 기준

```
파일이 작고, S3 전환 계획 없음          → Direct
로컬 개발 + 운영 S3 동일 인터페이스 원함 → Local Pre-signed + S3 Pre-signed
처음부터 S3 만 사용                     → S3 Pre-signed
```

## 디렉토리 구조

```
framework/storage/
├── storage.md                ← 전체 개요 (이 파일)
├── port/port.md              ← 포트 인터페이스 정의
├── local/
│   ├── direct.md             ← Direct 저장 구현
│   └── presigned.md          ← Local Pre-signed URL 구현
└── s3/s3.md                  ← AWS S3 Pre-signed URL 구현
```

## 공통 경로 패턴

```
{domain}/{year}/{month}/{day}/{UUID}/filename.ext

예: product/2024/01/15/550e8400.../image.png
```

## StoragePathUtil (공통)

경로 생성 알고리즘 공통 제공. 도메인별 메서드는 각 프로젝트에서 추가 선언.

```kotlin
@Service
class StoragePathUtil(
    private val properties: StorageProperties,   // Local 저장소 사용 시
) {
    // 도메인별 경로 생성 — 각 프로젝트에서 도메인에 맞게 추가
    fun getFooPath() = "${FOO}/${year()}/${month()}/${day()}/${UUID.randomUUID()}"
    fun getFooPrefix() = "${properties.prefix}${FOO}"

    companion object {
        const val FOO = "foo"     // 도메인 상수 (선두 슬래시 없음)

        fun year(): String = String.format("%d", TimeUtil.nowKST().year)
        fun month(): String = String.format("%02d", TimeUtil.nowKST().month.value)
        fun day(): String = String.format("%02d", TimeUtil.nowKST().dayOfMonth)
    }
}
```

> 경로 패턴: `{domain}/{year}/{month}/{day}/{UUID}/`
