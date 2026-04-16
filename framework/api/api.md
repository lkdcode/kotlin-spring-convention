# framework/api — API 공통 프레임워크

## 디렉토리 구조

```
framework/api/
├── api.md              ← 전체 개요 (이 파일)
├── response/           ← ApiResponse, ApiResponseCode
├── exception/          ← ApiException
├── advice/             ← ApiAdvice (@RestControllerAdvice)
├── client/             ← ApiWebClient (외부 API 호출)
└── dto/                ← 공통 DTO (RestoreDTO 등)
```

## 흐름

```
Controller
    → throw ApiException(ApiResponseCode.FOO_NOT_FOUND)
        → ApiAdvice.handleApiException()
            → ApiResponse.clientError(code, payload)
                → ResponseEntity 반환

외부 API 호출
    → ApiWebClient.getRequest(url, headers, responseType)
        → 4xx/5xx → throw ApiException(ApiResponseCode.WEB_CLIENT_IS_4XX_ERROR)
```
