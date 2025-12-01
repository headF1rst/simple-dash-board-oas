# Contract 모듈 - OpenAPI Generator 가이드

## 개요

`contract` 모듈은 **OpenAPI 스펙(YAML)**을 기반으로 서버 스텁 인터페이스와 DTO 모델 클래스를 자동 생성하는 **"계약 전용(Contract-First)"** 모듈입니다.

### Contract-First 개발이란?

전통적인 방식(Code-First)에서는 코드를 먼저 작성하고 API 문서를 나중에 생성합니다. 반면 **Contract-First** 방식은:

1. **OpenAPI 스펙**을 먼저 작성합니다 (API 계약 정의)
2. 스펙으로부터 **서버 인터페이스**와 **클라이언트 코드**를 자동 생성합니다
3. 개발자는 생성된 인터페이스를 **구현(implements)** 하기만 하면 됩니다

이 방식의 장점:
- API 설계가 구현보다 선행되어 **일관된 API 설계** 가능
- 프론트엔드/백엔드 팀이 **동일한 스펙**을 기준으로 병렬 개발 가능
- 스펙 변경 시 **컴파일 에러**로 불일치 즉시 감지

---

## 디렉토리 구조

```text
contract/
├── openapi.yaml                      # 메인 진입점 (다른 파일들을 $ref로 참조)
├── paths/                            # API 엔드포인트 정의
│   ├── articles.yaml                 # GET/POST /v1/articles
│   └── articles-by-id.yaml           # GET/PUT/DELETE /v1/articles/{id}
├── components/
│   ├── schemas/                      # 데이터 모델 (DTO)
│   ├── parameters/                   # 공통 파라미터
│   ├── responses/                    # 공통 응답
│   └── examples/                     # 예제 데이터
├── openapi-templates/spring/         # 커스텀 Mustache 템플릿
│   ├── api.mustache                  # API 인터페이스 템플릿
│   ├── generatedAnnotation.mustache  # @Generated 제거용
│   └── licenseInfo.mustache          # 라이선스 헤더
└── build.gradle                      # OpenAPI Generator 설정
```

---

## 코드 생성 방법

### 요구사항
- **JDK 17 이상** (Java 21 권장)

### 명령어

```bash
# 코드 생성만 실행
./gradlew :contract:openApiGenerate

# 전체 빌드 (코드 생성 + 컴파일)
./gradlew :contract:build

# 클린 후 재생성
./gradlew :contract:clean :contract:openApiGenerate
```

### 생성물 경로
- **Java 코드**: `contract/build/generated/openapi/src/main/java`
- **패키지 구조**:
  - `simple.board.contract.api` - API 인터페이스
  - `simple.board.contract.model` - DTO 모델 클래스

---

## Gradle 설정 상세 설명

### build.gradle 핵심 설정

```gradle
openApiGenerate {
    generatorName = 'spring'              // Spring 서버 스텁 생성
    inputSpec = '...'                     // OpenAPI 스펙 파일 경로
    templateDir = '...'                   // 커스텀 Mustache 템플릿 경로
    
    apiPackage = 'simple.board.contract.api'
    modelPackage = 'simple.board.contract.model'
    
    globalProperties = [
        apis           : '',              // API 인터페이스 생성
        models         : '',              // 모델 클래스 생성
        supportingFiles: ''               // 지원 파일 미생성
    ]
    
    configOptions = [
        interfaceOnly          : 'true',  // 인터페이스만 생성 (구현체 X)
        useTags                : 'true',  // 태그 기반 API 분리
        addGeneratedAnnotation : 'false', // @Generated 어노테이션 제거
        documentationProvider  : 'none',  // Swagger 어노테이션 제거
        useBeanValidation      : 'true',  // @Valid 등 검증 어노테이션 추가
        useJakartaEe           : 'true',  // Jakarta EE 사용 (javax → jakarta)
        skipDefaultInterface   : 'true'   // default 메서드 본문 제거
    ]
}
```

### 주요 옵션 설명

| 옵션 | 설명 |
|------|------|
| `interfaceOnly: true` | 인터페이스만 생성, 구현체는 직접 작성 |
| `documentationProvider: none` | `@Operation`, `@ApiResponse` 등 Swagger 어노테이션 제거 |
| `addGeneratedAnnotation: false` | `@Generated` 어노테이션 제거 |
| `skipDefaultInterface: true` | `default` 메서드의 `NOT_IMPLEMENTED` 본문 제거 |
| `useJakartaEe: true` | `javax.validation` → `jakarta.validation` |

---

## 커스텀 Mustache 템플릿

OpenAPI Generator는 **Mustache 템플릿**을 사용하여 코드를 생성합니다. 기본 템플릿은 Swagger 어노테이션과 장황한 코드를 포함하므로, 간결한 코드를 위해 커스텀 템플릿을 사용합니다.

### api.mustache - API 인터페이스 템플릿

**기본 템플릿의 문제점:**
```java
// 기본 템플릿 생성 결과 (장황함)
@Operation(summary = "게시글 목록 조회", ...)
@ApiResponses(value = { ... })
@RequestMapping(
    method = RequestMethod.GET,
    value = "/v1/articles",
    produces = { "application/json" }
)
default ResponseEntity<ArticleListResponse> listArticles(...) {
    return new ResponseEntity<>(HttpStatus.NOT_IMPLEMENTED);
}
```

**커스텀 템플릿 생성 결과 (간결함):**
```java
@GetMapping("/v1/articles")
ResponseEntity<ArticleListResponse> listArticles(...);
```

### 템플릿 파일 설명

| 파일 | 역할 |
|------|------|
| `api.mustache` | API 인터페이스 생성 - `@GetMapping`, `@PostMapping` 등 사용 |
| `generatedAnnotation.mustache` | 빈 파일로 `@Generated` 어노테이션 비활성화 |
| `licenseInfo.mustache` | 파일 상단 주석 |

---

## 서비스 모듈에서 사용하기

### 1. 의존성 추가

```gradle
// service/article/build.gradle
dependencies {
    implementation project(":contract")
}
```

### 2. 인터페이스 구현

```java
@RestController
@RequiredArgsConstructor
public class ArticleController implements ArticleApi {
    
    private final ArticleService articleService;
    
    @Override
    public ResponseEntity<Article> createArticle(ArticleRequest body) {
        Article created = articleService.create(body);
        return ResponseEntity.created(URI.create("/v1/articles/" + created.getId()))
                           .body(created);
    }
    
    @Override
    public ResponseEntity<ArticleListResponse> listArticles(
            Long boardId, Long writerId, Integer page, Integer size, String sort) {
        return ResponseEntity.ok(articleService.list(boardId, writerId, page, size, sort));
    }
    
    // ... 나머지 메서드 구현
}
```

### 3. 장점

- **컴파일 타임 검증**: 스펙과 구현의 불일치 시 컴파일 에러 발생
- **IDE 자동완성**: 인터페이스 메서드 자동 생성
- **일관된 API**: 모든 컨트롤러가 동일한 시그니처 준수

---

## 스펙 번들링

OpenAPI 스펙이 여러 파일로 분리되어 있을 때(`$ref` 사용), 코드 생성기가 상대 경로를 올바르게 해석하도록 **번들링** 과정이 필요합니다.

### 현재 방식: 단순 복사

`bundleOpenApi` 태스크가 스펙 파일들을 `build` 디렉토리로 복사합니다:

```
openapi.yaml      →  build/openapi.bundle.yaml
paths/            →  build/paths/
components/       →  build/components/
```

### 고급 옵션: 실제 번들러 사용

대규모 스펙에서는 실제 번들링 도구 사용을 권장합니다:

- **Redocly CLI**: `npx @redocly/cli bundle openapi.yaml -o openapi.bundle.yaml`
- **OpenAPI Generator CLI**: `openapi-generator-cli bundle -i openapi.yaml -o openapi.bundle.yaml`

---

## 의존성 스코프

contract 모듈은 **라이브러리**이므로, 모든 의존성을 `compileOnly`로 선언합니다:

```gradle
dependencies {
    compileOnly 'org.springframework:spring-web'
    compileOnly 'jakarta.validation:jakarta.validation-api'
    compileOnly 'com.fasterxml.jackson.core:jackson-annotations'
    compileOnly 'org.openapitools:jackson-databind-nullable:0.2.6'
}
```

실제 런타임 의존성은 이 모듈을 사용하는 **서비스 모듈**에서 제공됩니다.

---

## 트러블슈팅

### Q: `Unable to load RELATIVE ref` 에러가 발생합니다

`$ref`로 참조하는 파일들이 올바른 위치에 있는지 확인하세요. `bundleOpenApi` 태스크가 정상 실행되었는지 확인합니다.

### Q: 어색한 모델명(예: `Articlesid`)이 생성됩니다

파일명에 특수문자(`@`, `{`, `}`)가 포함되면 이상한 이름이 생성될 수 있습니다.
- 해결: `articles@{id}.yaml` → `articles-by-id.yaml`로 변경
- 또는 스키마에 명시적인 `title` 속성 부여

### Q: `@Generated` 어노테이션이 계속 생성됩니다

1. `configOptions`에 `addGeneratedAnnotation: 'false'` 확인
2. `generatedAnnotation.mustache` 파일이 빈 파일인지 확인
3. `./gradlew :contract:clean :contract:openApiGenerate` 실행

---

## 참고 자료

- [OpenAPI Generator 공식 문서](https://openapi-generator.tech/)
- [OpenAPI 3.0 스펙](https://spec.openapis.org/oas/v3.0.3)
- [Mustache 템플릿 문법](https://mustache.github.io/mustache.5.html)
