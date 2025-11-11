# simple-dash-board-oas

## 디렉토리 구조

```text
api/
├── openapi.yaml                 # 메인 진입점 파일
├── paths/                       # 각 엔드포인트별 파일
│   ├── articles.yaml           # GET /v1/articles
│   ├── articles@{id}.yaml      # GET/PUT/DELETE /v1/articles/{id}
│   └── articles@create.yaml    # POST /v1/articles
├── components/
│   ├── schemas/                # 데이터 모델
│   │   ├── Article.yaml
│   │   ├── ArticleListResponse.yaml
│   │   ├── ArticleRequest.yaml
│   │   ├── Pagination.yaml
│   │   └── ErrorResponse.yaml
│   ├── parameters/             # 공통 파라미터
│   │   ├── PageQuery.yaml
│   │   ├── SizeQuery.yaml
│   │   └── SortQuery.yaml
│   ├── responses/              # 공통 응답
│   │   ├── 400BadRequest.yaml
│   │   ├── 401Unauthorized.yaml
│   │   ├── 404NotFound.yaml
│   │   └── 500ServerError.yaml
│   └── examples/               # 예제 데이터
│       ├── ArticleExample.yaml
│       └── ErrorExample.yaml
└── README.md

```
