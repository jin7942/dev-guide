## 1. 디렉터리 구조 원칙

```
src/
├── domain/             # 도메인 단위 모듈
│   └── xxx/            # ex) server/, container/
│       ├── controller/
│       ├── service/
│       ├── dto/
│       └── route/
├── common/
│   ├── types/          # ResponseVo, HttpStatus, 등 공통 타입 정의
│   ├── utils/          # 유틸리티 함수
│   ├── error/          # CustomError 등 공통 에러 정의
│   └── middleware/     # asyncHandler, errorHandler 등 미들웨어
└── main.ts / app.ts

```

**이유**:

-   기능별, 책임별 디렉토리 분리를 통해 유지보수와 가독성 향상
-   도메인 주도 설계에 근접한 구조 확보

## 2. 파일 네이밍 규칙

-   **컨트롤러/서비스/라우터 파일**: `kebab-case.role.ts` 형식
    -   ex) `server-http.controller.ts`, `container-http.service.ts`
-   **유틸/헬퍼 함수**: `도메인-유형.ts`
    -   ex) `create-util.ts`, `docker-util.ts`
-   **클래스 단독 파일 (ex: CustomError)**: `PascalCase.ts`
    -   ex) `CustomError.ts`
-   **enum, types**: `snake-case.enum.ts`, `xxx.vo.ts`, `xxx.dto.ts`

**이유**:

-   일관된 정렬 및 자동 탐색이 쉬움
-   직관적인 기능 구분 가능

## 3. 응답 포맷

### HTTP

```typescript
{
    /**
     * 공통 HTTP 응답 포맷
     * - {success: boolean, message: string, data: T}
     * @template T 실제 응답 데이터 타입
     */
    export interface ResponseVo<T> {
        /** 요청 성공 여부 (true/false) */
        success: boolean;

        /** 결과 메시지 (예: "조회 성공", "서버 오류" 등) */
        message: string;

        /** 실제 응답 데이터 */
        data: T; // 실패시 { code: HttpStatusCode }
    }
}
```

### WebSocket

```typescript
/**
 * WebSocket 전용 응답 포맷
 * @template T 실제 실시간 데이터 타입
 */
export interface WsVo<T> {
    /** 데이터 종류 (예: 'cpu', 'memory', 'network' 등) */
    type: string;

    /** 데이터를 보낸 호스트명 */
    hostname: string;

    /** 데이터 생성 시각 (ISO 8601) */
    timestamp: string;

    /** 실시간 전송할 실제 데이터 */
    data: T;
}
```

-   HTTP 응답 생성은 `createResponseVo(success, message, data?)` 유틸 함수 사용
-   WebSocket 응답 생성은 `createWsResponseVo(type, hostname, data)` 유틸 함수 사용

**이유**:

-   일관된 포멧을 제공함으로서 가독성 증가 및 협업 개발자와 구조 통일
-   프론트엔드 연동시, 일관된 데이터 제공

## 4. 에러 처리

-   **모든 HTTP 요청은 라우터에서 `asyncHandler()`로 감싸야 한다**
-   **컨트롤러/서비스에서는 반드시 `throw new CustomError(HttpStatus.BAD_REQUEST, 'msg')` 형식으로 에러 발생**
-   전역 에러 핸들러가 `{ success: false, message, data: { code } }` 포맷으로 통일 응답

```typescript
// server-http.route.ts
serverRouter.get('/info', asyncHandler(serverController.getSysInfo));
```

```typescript
// CustomError.ts
export class CustomError extends Error {
    constructor(public code: HttpStatus, message: string) {
        super(message);
        this.name = 'CustomError';
    }
}
```

```typescript
// error-handler.ts
if (err instanceof CustomError) {
    res.status(err.code).json(createResponseVo(false, err.message, { code: err.code }));
}
```

**이유**:

-   try-catch 중복 제거
-   에러 응답 포맷 일관화
-   모든 에러 흐름을 전역에서 통제 가능

## 5. HTTP 상태 코드 관리

-   HTTP 상태 코드는 enum으로 관리한다:

```typescript
export enum HttpStatus {
    BAD_REQUEST = 400,
    UNAUTHORIZED = 401,
    FORBIDDEN = 403,
    NOT_FOUND = 404,
    CONFLICT = 409,
    INTERNAL_SERVER_ERROR = 500,
    SERVICE_UNAVAILABLE = 503,
}
```

-   `code` 필드는 반드시 `HttpStatus` enum 값만 허용

**이유**:

-   의미 없는 숫자 남용 방지
-   자동완성 및 오타 방지
-   상태코드 의미를 명시적으로 드러냄 (ex: `BAD_REQUEST` vs 400)

---

## 6. WebSocket 처리 규칙

-   주기 송신은 `setWsIntervalSender()` 유틸로 처리
-   이벤트 기반 송신은 `setWsStreamSender()`유틸로 처리
-   내부에서 `try-catch`로 에러 감지 시 `ws.close()` 및 interval 해제
-   WebSocket은 asyncHandler 패턴을 쓸 수 없으므로 함수 내부에서 명시적으로 에러 처리

**이유**:

-   WebSocket은 미들웨어 체인이 없어 asyncHandler 방식 적용 불가
-   에러가 발생하면 스트림을 종료하고 연결도 닫는 것이 안전한 기본 동작

---

## 7. 네이밍 규칙 요약

| 요소            | 형식                 | 예시                        |
| --------------- | -------------------- | --------------------------- |
| 디렉터리        | kebab-case           | `server`, `container`       |
| 컨트롤러/서비스 | kebab-case + suffix  | `server-http.controller.ts` |
| 클래스 파일     | PascalCase           | `CustomError.ts`            |
| DTO             | PascalCase           | `ContainerDto`              |
| VO              | PascalCase           | `ResponseVo`                |
| enum 파일       | kebab-case.enum.ts   | `http-status.enum.ts`       |
| 유틸 함수 파일  | 기능 중심 kebab-case | `create-response.ts`        |

**이유**:

-   탐색/정렬 용이
-   어떤 역할의 파일인지 파일명만 보고 판단 가능
-   타입스크립트 커뮤니티와 호환

## 8. 주석

주석은 다음과 같은 형태로 작성한다.

-   tsDoc 스타일을 준수하되, 컨트롤러는 예외적으로 리턴타입을 표시한다.
-   컨트롤러에서 `POST` 요청을 처리할 시, `@RequestBody`로 타입을 명시한다
-   컨트롤러는 `@route`를 이용해 요청 경로를 명시한다.
-   파라미터가 있을 경우, 주석에 명시한다.

컨트롤러 예시

```typescript
    /**
     * 로그 파일 조회 API
     * - 파라미터로 파일 전체 경로를 요청
     * - 경로노출 최소화를 위해 POST로 처리
     *
     * @route POST /api/log/file
     * @requestBody {LogPathDto}
     * @returns {LogFileContentVo} 해당 파일의 전체 텍스트를 담은 JSON 객체
     */
    public getLogFile = async (req: Request, res: Response): Promise<void> => {
        const filePathDto: LogPathDto = req.body;
        res.status(200).json(createResponseVo(true, '파일 조회 성공', resData));
    };
```

기타 함수 예시

```typescript
/**
 * 비동기 컨트롤러 핸들러
 * - 컨트롤러 호출을 감싸서 에러를 next()로 자동 전달하는 함수
 * - 에러 핸들링은 /common/error/error-handler.ts 에서 처리
 *
 * @param handler - Express RequestHandler 함수 (비동기)
 * @returns 에러를 next로 전달하는 래핑 함수
 */
export const asyncHandler =
    (handler: RequestHandler): RequestHandler =>
    (req: Request, res: Response, next: NextFunction) => {
        Promise.resolve(handler(req, res, next)).catch(next);
    };
```

## 9. 기타

-   주석은 가능한 한 모두 작성한다 (`@route`, `@returns`, `@param`, 등 포함)
-   응답 형식은 통일된 유틸 함수로 생성하며, 직접 JSON 객체 생성은 지양한다
-   에러 응답도 통일된 `CustomError` → `errorHandler` 흐름만 사용한다

**이유**:

-   문서화 수준의 코드 유지
-   협업자 및 미래의 나를 위한 명확한 기준 제공
