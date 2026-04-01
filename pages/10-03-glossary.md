# 레슨 10.3: n8n 용어 사전

---

n8n과 자동화에서 자주 등장하는 용어들을 알파벳순(가나다순)으로 정리했습니다.

---

## ㄱ

**개인정보 (PII, Personally Identifiable Information)**
이름, 이메일, 전화번호 등 개인을 특정할 수 있는 정보. n8n 워크플로우에서 처리 시 개인정보보호법 준수 필요.

**고정 데이터 (Static Data)**
워크플로우 실행 간에 유지되는 데이터. `$workflow.getStaticData('global')` 또는 `$workflow.getStaticData('node')`로 접근.

**그래프 (Graph)**
n8n 워크플로우의 노드와 연결의 시각적 표현. n8n은 방향성 비순환 그래프(DAG) 구조 사용.

---

## ㄴ

**노드 (Node)**
n8n 워크플로우의 기본 단위. 각 노드는 특정 작업(이메일 발송, API 호출, 데이터 변환 등)을 수행.

**노드 타입**
- 트리거 노드: 워크플로우를 시작하는 노드
- 액션 노드: 실제 작업을 수행하는 노드
- 코어 노드: IF, Switch, Merge 등 흐름 제어 노드

---

## ㄷ

**데이터 매핑 (Data Mapping)**
이전 노드의 데이터를 다음 노드의 입력으로 연결하는 과정. 표현식(Expression)으로 구현.

**드래그 앤 드롭 (Drag and Drop)**
n8n UI에서 노드를 마우스로 끌어다 배치하는 방식.

---

## ㄹ

**로그 (Log)**
워크플로우 실행 중 기록되는 정보. Code 노드에서 `console.log()`로 기록.

---

## ㅁ

**메모리 (Memory)**
AI 에이전트가 이전 대화를 기억하는 기능. PostgreSQL Chat Memory, Buffer Memory 등.

**멀티 에이전트 (Multi-Agent)**
여러 AI 에이전트가 역할을 나누어 협력하는 시스템.

**모듈 (Module)**
재사용 가능한 단위로 분리된 워크플로우나 코드. 서브 워크플로우로 구현.

---

## ㅂ

**배치 처리 (Batch Processing)**
대량의 데이터를 한번에 처리하지 않고 작은 단위(배치)로 나누어 처리하는 방식. `Split In Batches` 노드 사용.

**벡터 데이터베이스 (Vector Database)**
텍스트를 수치 벡터로 변환하여 저장하고, 유사도 검색이 가능한 데이터베이스. Pinecone, Qdrant, Supabase+pgvector 등.

**변환 (Transformation)**
데이터의 형태나 구조를 바꾸는 작업. Code 노드나 Set 노드로 구현.

**병렬 처리 (Parallel Processing)**
여러 작업을 동시에 실행하여 속도를 높이는 방식.

---

## ㅅ

**서브 워크플로우 (Sub-Workflow)**
다른 워크플로우에서 호출할 수 있는 독립적인 워크플로우. `Execute Workflow` 노드로 호출.

**스케줄 트리거 (Schedule Trigger)**
설정된 시간(크론 표현식)에 자동으로 워크플로우를 시작하는 트리거.

**실행 (Execution)**
워크플로우가 한 번 실행되는 단위. 성공/실패 이력이 저장됨.

---

## ㅇ

**액티브 (Active)**
워크플로우가 활성화된 상태. 트리거 조건이 충족되면 자동으로 실행됨.

**에이전트 (Agent)**
LLM(AI 언어모델)이 도구(Tools)를 사용하여 스스로 판단하고 행동하는 시스템.

**에러 트리거 (Error Trigger)**
다른 워크플로우에서 오류 발생 시 자동으로 실행되는 특수 트리거. 오류 알림에 사용.

**에스컬레이션 (Escalation)**
자동화가 해결하지 못하는 상황을 인간에게 전달하는 과정.

**연결 (Connection)**
두 노드를 연결하는 선. 데이터 흐름의 방향을 나타냄.

**오케스트레이터 (Orchestrator)**
멀티 에이전트 시스템에서 전체 작업을 계획하고 하위 에이전트에게 배분하는 중앙 에이전트.

---

## ㅈ

**임베딩 (Embedding)**
텍스트를 수치 벡터로 변환하는 기술. RAG 시스템에서 문서 검색에 활용.

**자격증명 (Credentials)**
API 키, 비밀번호, OAuth 토큰 등 외부 서비스 접근에 필요한 인증 정보. n8n에서 암호화하여 저장.

**조건 분기 (Conditional Branching)**
특정 조건에 따라 워크플로우 흐름을 다르게 나누는 것. `IF` 노드 또는 `Switch` 노드 사용.

**주입 (Injection)**
워크플로우 실행 시 외부 데이터를 입력으로 전달하는 것. 웹훅 페이로드, 트리거 데이터 등.

---

## ㅊ

**청크 (Chunk)**
긴 텍스트를 처리 가능한 단위로 나눈 조각. RAG 시스템에서 문서를 300-500 토큰씩 분할.

**출력 (Output)**
노드가 처리 후 다음 노드에 전달하는 데이터.

---

## ㅋ

**크론 표현식 (Cron Expression)**
스케줄을 정의하는 문자열 형식. `0 9 * * 1` = 매주 월요일 오전 9시.

**크리덴셜 (Credential)**
자격증명의 영문 표기. n8n에서 외부 서비스 인증 정보 저장소.

---

## ㅌ

**타임아웃 (Timeout)**
지정된 시간 내에 작업이 완료되지 않으면 중단하는 설정.

**테스트 모드 (Test Mode)**
워크플로우를 수동으로 한 번 실행하여 테스트하는 방식. 결과를 즉시 확인 가능.

**토큰 (Token)**
LLM에서 텍스트를 처리하는 기본 단위. 대략 4글자 ≈ 1 토큰. API 비용의 기준.

**트리거 (Trigger)**
워크플로우를 시작시키는 이벤트 또는 노드. 웹훅, 스케줄, 폴링 등 다양한 유형.

---

## ㅍ

**페이로드 (Payload)**
웹훅이나 API 요청/응답에 담긴 실제 데이터.

**폴링 (Polling)**
변경 사항 감지를 위해 주기적으로 서버를 확인하는 방식. 웹훅과 대조됨.

**표현식 (Expression)**
n8n에서 동적 값을 만들기 위한 문법. `{{ $json.fieldName }}` 형식.

---

## ㅎ

**할루시네이션 (Hallucination)**
AI 언어모델이 사실이 아닌 정보를 확신을 가지고 생성하는 현상. RAG로 완화 가능.

**헬스체크 (Health Check)**
시스템이 정상 작동 중인지 주기적으로 확인하는 작업.

**환경 변수 (Environment Variable)**
시스템 레벨에서 설정하는 변수. n8n에서 `N8N_PORT`, `DB_TYPE` 등으로 서버 설정 제어.

---

## 영문 약어

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| API | Application Programming Interface | 시스템 간 통신 인터페이스 |
| CI/CD | Continuous Integration/Deployment | 지속적 통합 및 배포 |
| CRUD | Create Read Update Delete | 데이터 기본 조작 |
| ETL | Extract Transform Load | 데이터 추출-변환-적재 |
| JSON | JavaScript Object Notation | 데이터 교환 형식 |
| LLM | Large Language Model | 대형 언어 모델 (GPT, Claude 등) |
| MFA | Multi-Factor Authentication | 다중 인증 |
| OAuth | Open Authorization | 표준 인증 프로토콜 |
| RAG | Retrieval-Augmented Generation | 검색 증강 생성 |
| REST | Representational State Transfer | API 설계 원칙 |
| SDK | Software Development Kit | 개발 도구 모음 |
| SQL | Structured Query Language | 데이터베이스 질의 언어 |
| SSL/TLS | Secure Sockets Layer/Transport Layer Security | 암호화 통신 |
| UI | User Interface | 사용자 인터페이스 |
| URL | Uniform Resource Locator | 웹 주소 |
| UTC | Coordinated Universal Time | 협정 세계시 (UTC+9 = 한국시간) |
| UUID | Universally Unique Identifier | 전역 고유 식별자 |
| VPN | Virtual Private Network | 가상 사설망 |
| YAML | Yet Another Markup Language | 설정 파일 형식 |

---

## 마치며

이 용어 사전이 n8n을 공부하고 사용하면서 모르는 단어가 나올 때 유용한 참고서가 되길 바랍니다.

자동화의 세계는 계속 발전하고 있습니다. 새로운 개념과 도구들이 등장하지만, 이 책에서 배운 **기본 원리**는 변하지 않습니다:

> 반복적인 작업을 자동화하고,
> AI를 활용하여 더 스마트하게 일하고,
> 절약된 시간으로 더 중요한 일에 집중하세요.

**n8n으로 자동화하는 모든 분들의 성공을 응원합니다!** 🚀

---

## 문의 및 커뮤니티

![n8n AI 자동화 가이드 표지](../assets/cover.png)

- 이메일 문의: [leemanrank@gmail.com](mailto:leemanrank@gmail.com)
- 인공지능 정보공유 단톡방 참여: [https://open.kakao.com/o/s4OEqBai](https://open.kakao.com/o/s4OEqBai)
