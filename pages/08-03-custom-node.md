# 레슨 8.3: 커스텀 노드 개발

---

## 커스텀 노드가 필요한 경우

n8n은 400개 이상의 내장 노드를 제공하지만, 특수한 상황에서는 직접 노드를 만들어야 합니다:

- 내장 노드가 없는 독자적인 API 또는 내부 시스템 연동
- 복잡한 로직을 매번 Code 노드로 작성하기 불편할 때
- 팀 내에서 공통으로 사용하는 기능을 노드로 패키징할 때
- n8n 커뮤니티에 오픈소스 노드로 기여하고 싶을 때

---

## 개발 환경 설정

### 필수 도구

```bash
# Node.js 18+ 필요
node --version  # v18.x.x 이상

# n8n 커스텀 노드 개발 도구
npm install -g @n8n/node-dev

# 프로젝트 생성
n8n-node-dev new
```

### 프로젝트 구조

```
my-n8n-node/
├── nodes/
│   └── MyService/
│       ├── MyService.node.ts    # 노드 로직
│       ├── MyService.node.json  # 노드 설명
│       └── myservice.svg        # 노드 아이콘
├── credentials/
│   └── MyServiceApi.credentials.ts
├── package.json
└── tsconfig.json
```

---

## 기본 노드 구조

```typescript
// nodes/MyService/MyService.node.ts
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
  NodeOperationError,
} from 'n8n-workflow';

export class MyService implements INodeType {
  description: INodeTypeDescription = {
    // 노드 기본 정보
    displayName: 'My Service',
    name: 'myService',
    icon: 'file:myservice.svg',
    group: ['transform'],
    version: 1,
    description: '내부 서비스 연동 노드',

    // 노드 입력/출력
    inputs: ['main'],
    outputs: ['main'],

    // 크리덴셜
    credentials: [
      {
        name: 'myServiceApi',
        required: true,
      },
    ],

    // 노드 속성 (UI에서 설정하는 옵션들)
    properties: [
      {
        displayName: '동작',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        options: [
          {
            name: '조회',
            value: 'get',
            description: '항목 조회',
          },
          {
            name: '생성',
            value: 'create',
            description: '새 항목 생성',
          },
        ],
        default: 'get',
      },
      {
        displayName: '항목 ID',
        name: 'itemId',
        type: 'string',
        required: true,
        displayOptions: {
          show: {
            operation: ['get'],
          },
        },
        default: '',
        description: '조회할 항목의 ID',
      },
    ],
  };

  // 실제 실행 로직
  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
      const operation = this.getNodeParameter('operation', i) as string;
      const credentials = await this.getCredentials('myServiceApi');

      if (operation === 'get') {
        const itemId = this.getNodeParameter('itemId', i) as string;

        // API 호출
        const response = await this.helpers.request({
          method: 'GET',
          url: `${credentials.baseUrl}/items/${itemId}`,
          headers: {
            Authorization: `Bearer ${credentials.apiKey}`,
          },
          json: true,
        });

        returnData.push({
          json: response,
        });
      }
    }

    return [returnData];
  }
}
```

---

## 크리덴셜 정의

```typescript
// credentials/MyServiceApi.credentials.ts
import {
  ICredentialType,
  INodeProperties,
} from 'n8n-workflow';

export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'My Service API';
  documentationUrl = 'https://docs.myservice.com/api';

  properties: INodeProperties[] = [
    {
      displayName: 'API 키',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },  // 비밀번호 마스킹
      default: '',
    },
    {
      displayName: '서버 URL',
      name: 'baseUrl',
      type: 'string',
      default: 'https://api.myservice.com',
    },
  ];
}
```

---

## 실전 예시: 사내 ERP 연동 노드

```typescript
// 사내 ERP API 노드 예시
properties: [
  {
    displayName: '모듈',
    name: 'module',
    type: 'options',
    options: [
      { name: '직원 관리', value: 'employees' },
      { name: '재고 관리', value: 'inventory' },
      { name: '주문 관리', value: 'orders' },
    ],
    default: 'employees',
  },
  {
    displayName: '작업',
    name: 'action',
    type: 'options',
    options: [
      { name: '목록 조회', value: 'list' },
      { name: '상세 조회', value: 'detail' },
      { name: '업데이트', value: 'update' },
    ],
    default: 'list',
  },
  // 모듈에 따라 다른 필드 표시
  {
    displayName: '직원 번호',
    name: 'employeeId',
    type: 'string',
    displayOptions: {
      show: {
        module: ['employees'],
        action: ['detail', 'update'],
      },
    },
    default: '',
  },
]
```

---

## 노드 빌드 및 설치

```bash
# TypeScript 컴파일
npm run build

# n8n에 로컬 노드 연결 (개발용)
npm link

# n8n 환경변수 설정
export N8N_CUSTOM_EXTENSIONS="/path/to/my-n8n-node"

# n8n 재시작 후 노드 확인
```

### Docker 환경에서 커스텀 노드 사용

```yaml
# docker-compose.yml
services:
  n8n:
    image: n8nio/n8n
    volumes:
      - ./my-n8n-node:/home/node/.n8n/custom
    environment:
      - N8N_CUSTOM_EXTENSIONS=/home/node/.n8n/custom
```

---

## npm 패키지로 배포

```json
// package.json
{
  "name": "n8n-nodes-myservice",
  "version": "1.0.0",
  "description": "My Service n8n 노드",
  "keywords": [
    "n8n-community-node-package"
  ],
  "main": "index.js",
  "n8n": {
    "n8nNodesApiVersion": 1,
    "credentials": [
      "dist/credentials/MyServiceApi.credentials.js"
    ],
    "nodes": [
      "dist/nodes/MyService/MyService.node.js"
    ]
  }
}
```

```bash
# npm에 배포
npm publish

# n8n에서 커뮤니티 노드 설치
# Settings → Community Nodes → Install
# 패키지명: n8n-nodes-myservice
```

---

## 고급 기능

### 웹훅 지원 노드

```typescript
// 웹훅 트리거 노드
export class MyWebhookNode implements INodeType {
  description: INodeTypeDescription = {
    // ...
    inputs: [],
    outputs: ['main'],
    webhooks: [
      {
        name: 'default',
        httpMethod: 'POST',
        responseMode: 'onReceived',
        path: 'webhook',
      },
    ],
  };

  async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const body = this.getBodyData();
    return {
      workflowData: [[{ json: body as IDataObject }]],
    };
  }
}
```

### 파일 처리 노드

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();

  for (let i = 0; i < items.length; i++) {
    // 바이너리 데이터 읽기
    const binaryData = this.helpers.assertBinaryData(i, 'data');
    const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');

    // 처리 후 새 바이너리 데이터 반환
    const processedBuffer = processFile(buffer);
    const newBinaryData = await this.helpers.prepareBinaryData(
      processedBuffer,
      'processed.pdf'
    );

    returnData.push({
      json: { processed: true },
      binary: { data: newBinaryData },
    });
  }
}
```

---

## 핵심 요약

- 커스텀 노드 = TypeScript로 작성하는 n8n 확장 기능
- `n8n-node-dev` 도구로 프로젝트 스캐폴딩
- 노드 속성 → UI 설정, execute() → 실제 로직
- 크리덴셜 분리로 API 키 안전하게 관리
- npm 패키지로 배포하여 팀 또는 커뮤니티와 공유
- Docker 볼륨 마운트로 자체 호스팅 환경에 설치

**다음 레슨**: n8n 성능 최적화로 대용량 데이터를 처리합니다.
