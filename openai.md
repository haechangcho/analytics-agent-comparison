# OpenAI 내부 데이터 에이전트 살펴보기
**Bonnie Xu, Aravind Suresh, Emma Tang 작성**

*문서 내용 듣기 15:16*

### 목차
*   [맞춤형 도구가 필요했던 이유](#맞춤형-도구가-필요했던-이유)
*   [작동 방식](#작동- 방식)
*   [맥락이 전부입니다](#맥락이-전부입니다)
*   Built to think and work like a teammate
*   Moving fast without breaking trust
*   Agent security
*   Lessons learned
*   Same vision, new tools

---

데이터는 시스템이 학습하는 방식, 제품이 진화하는 방식, 기업이 의사결정을 내리는 방식을 좌우합니다. 하지만 빠르고 정확하며 올바른 맥락을 갖춘 답을 얻는 일은 종종 필요 이상으로 어렵습니다. OpenAI가 확장됨에 따라 이를 더 쉽게 만들기 위해, 우리는 자체 맞춤형 내부 AI 데이터 에이전트를 구축했으며, 이 에이전트는 자체 플랫폼 전반을 탐색하고 추론합니다.

이 에이전트는 외부에 제공되지 않는 내부 전용 맞춤형 툴로, OpenAI의 데이터, 권한, 워크플로에 맞게 설계되었습니다. 우리는 이 에이전트를 어떻게 구축하고 활용했는지, 그리고 AI가 팀 전반의 일상 업무를 실질적으로 지원하는 사례를 어떻게 만들어내는지를 소개하고 있습니다. 이를 구축하고 운영하는 데 사용한 OpenAI 툴(Codex, GPT‑5 플래그십 모델, Evals API, Embeddings API)은 전 세계 개발자에게 제공되는 것과 동일한 툴입니다.

이 데이터 에이전트는 직원들이 며칠이 아니라 몇 분 만에 질문에서 인사이트로 이동할 수 있게 해줍니다. 이를 통해 데이터 팀뿐만 아니라 모든 부서에서 데이터 접근과 정교한 분석의 진입 장벽이 낮아집니다. 현재 OpenAI의 엔지니어링, 데이터 사이언스, Go-To-Market, 재무, 연구 팀 전반에서 이 에이전트를 활용해 영향력이 큰 데이터 질문에 답하고 있습니다. 예를 들어, 자연어라는 직관적인 형식을 통해 출시 성과를 평가하거나 비즈니스 상태를 이해하는 데 도움을 줄 수 있습니다. 이 에이전트는 Codex 기반의 테이블 수준 지식을 제품 및 조직 맥락과 결합합니다. 지속적으로 학습하는 메모리 시스템 덕분에, 사용할수록 성능이 향상됩니다.

> *2025년 10월 6일의 ChatGPT WAU를 DevDay 2023과 비교해 달라는 사용자 질문의 스크린샷입니다. 에이전트는 2025년에 약 8억 WAU, 2023년에 약 1억 WAU를 보고하며, +7억 증가와 약 8배 성장에 대한 설명을 덧붙입니다.*

이 글에서는 맞춤형 AI 데이터 에이전트가 필요했던 이유, 코드로 보강된 데이터 맥락과 자기 학습이 왜 유용한지, 그리고 그 과정에서 얻은 교훈을 살펴봅니다.

---

## 맞춤형 도구가 필요했던 이유
OpenAI의 데이터 플랫폼은 엔지니어링, 제품, 연구 전반에서 활동하는 3,500명 이상의 내부 사용자를 지원하며, 7만 개 데이터셋에 걸쳐 600페타바이트 이상의 데이터를 다룹니다. 이 정도 규모에서는 올바른 테이블을 찾는 것만으로도 분석에서 가장 시간이 많이 드는 작업 중 하나가 됩니다.

한 내부 사용자는 이렇게 말했습니다.

> “서로 매우 비슷한 테이블이 너무 많아서, 차이점과 어떤 것을 사용해야 하는지 파악하는 데 엄청난 시간이 소요됩니다. 어떤 테이블은 로그아웃된 사용자를 포함하고, 어떤 테이블은 포함하지 않습니다. 일부는 필드가 겹쳐서 무엇이 무엇인지 구분하기가 어렵습니다.”

올바른 테이블을 선택했더라도 정확한 결과를 도출하는 것은 여전히 어렵습니다. 분석가는 변환과 필터가 올바르게 적용되었는지 확인하기 위해 테이블 데이터와 테이블 간 관계를 이해해야 합니다. 다대다 조인, 필터 푸시다운 오류, 처리되지 않은 null 값과 같은 일반적인 실패 요인은 결과를 조용히 무효화할 수 있습니다. OpenAI와 같은 규모에서는 분석가가 SQL 문법이나 쿼리 성능 디버깅에 시간을 쏟기보다는, 지표 정의, 가정 검증, 데이터 기반 의사결정에 집중해야 합니다.

> *고객 지역 데이터를 조인하고 주문-월 필드를 도출하며, 주문 수, 총매출, 세금 포함 매출, 평균 배송 완료까지의 일수와 같은 월별 집계를 계산하는 두 개의 CTE(order_enriched, monthly_segment)를 정의한 SQL 코드 스크린샷입니다. 이 SQL 문은 180줄이 넘습니다. 올바른 테이블을 조인하고 적절한 열을 조회하고 있는지 판단하기가 쉽지 않습니다.*

---

## 작동 방식
이제 에이전트가 무엇인지, 어떻게 맥락을 구성하는지, 그리고 어떻게 스스로 개선되는지 살펴보겠습니다.

우리의 에이전트는 **GPT‑5.2**로 구동되며 OpenAI의 데이터 플랫폼을 기반으로 추론하도록 설계되었습니다. 이 에이전트는 직원들이 이미 일하고 있는 모든 환경에서 사용할 수 있습니다. Slack 에이전트, 웹 인터페이스, IDE 내부, MCP를 통한 Codex CLI, 그리고 MCP 커넥터를 통한 OpenAI 내부 ChatGPT 앱에서 바로 이용할 수 있습니다.

> *“데이터 에이전트가 작동하는 방식”이라는 제목의 다이어그램. 진입점인 Agent-UI, Local Agent-MCP, Remote Agent-MCP, Slack Agent가 Agent-API로 연결됩니다. 이 API는 내부 데이터 지식과 회사 맥락에 연결되고, 데이터 웨어하우스 및 플랫폼 소스와 동기화되며, Agent-MCP를 통해 GPT-5.2 모델과 요청을 주고받습니다.*

사용자는 일반적으로 여러 차례의 수동 탐색이 필요한 복잡하고 개방적인 질문을 할 수 있습니다. 테스트 데이터 세트를 사용하는 다음 예시 프롬프트를 살펴보세요. “뉴욕시 택시 이동에서 일반적인 이동 시간과 최악의 이동 시간 간 차이가 가장 큰, 가장 신뢰할 수 없는 픽업-드롭오프 ZIP 쌍은 무엇이며, 그 변동성은 언제 발생하나요?”

에이전트는 질문 이해부터 데이터 탐색, 쿼리 실행, 결과 종합까지 분석 전 과정을 처리합니다.

*   **분석**
*   **데이터**
*   **쿼리**
*   **그래프**

> *뉴욕시 택시 픽업→드롭오프 ZIP 쌍 중 가장 “신뢰할 수 없는” 조합이 무엇인지 묻는 사용자 화면의 스크린샷입니다. 에이전트는 samples.nyctaxi.trips에서 약 21,000 건의 이동 데이터를 사용한다고 설명합니다. 일반적인 경우(p50)와 최악의 경우(p95)를 정의하고 필터를 적용하며, 각 ZIP 쌍에서 가장 오래 걸린 이동이 언제 발생했는지를 식별하는 방식을 설명합니다.*

질문에 대한 에이전트의 응답입니다.

이 에이전트의 강력한 장점 중 하나는 문제를 추론하는 방식입니다. 정해진 스크립트를 따르기보다는, 에이전트가 스스로 진행 상황을 평가합니다. 중간 결과가 잘못된 것처럼 보이면(예: 잘못된 조인이나 필터로 인해 행이 0개인 경우), 에이전트는 원인을 분석하고 접근 방식을 조정해 다시 시도합니다. 이 과정 전반에서 에이전트는 전체 맥락을 유지하며, 각 단계의 학습 내용을 다음 단계로 이어갑니다. 이러한 폐쇄형 자기 학습 프로세스는 반복 작업을 사용자에서 에이전트로 옮겨, 수동 워크플로보다 더 빠르고 일관되게 높은 품질의 분석을 가능하게 합니다.

> *뉴욕시 택시 이동 시간 분석을 위한 AI 에이전트의 단계별 계획을 보여주는 작업 워크플로 스크린샷입니다. 목표, 내부 검색, 스키마 점검, 코드 스니펫, p50/p95 차이 계산, 신뢰할 수 없는 ZIP 쌍 식별, SQL 쿼리 계획에 대한 사고 과정을 포함합니다.*
> *뉴욕시 택시 픽업–드롭오프 쌍 중 가장 신뢰할 수 없는 조합을 식별하기 위한 에이전트의 추론 과정.*

이 에이전트는 데이터 탐색, SQL 실행, 노트북 및 보고서 게시까지 전체 분석 워크플로를 포괄합니다. 사내 지식을 이해하고 외부 정보를 웹 검색할 수 있으며, 사용 경험과 메모리를 통해 시간이 지날수록 개선됩니다.

---

## 맥락이 전부입니다
고품질의 답변은 풍부하고 정확한 맥락에 달려 있습니다. 맥락이 없으면 아무리 강력한 모델이라도 사용자 수를 크게 잘못 추정하거나 내부 용어를 오해하는 등의 오류를 낼 수 있습니다.

> *사용자가 “지난 30일 동안 ChatGPT 이미지 생성의 로그인 DAU는 얼마였나요?”라고 묻는 화면의 스크린샷으로, 아래 상태 줄에는 에이전트가 “22분 41초 동안 작업 중”으로 표시되어 장시간 실행 중인 쿼리임을 나타냅니다.*
> **메모리가 없어 효과적으로 쿼리하지 못하는 에이전트.**

> *사용자가 “지난 30일 동안 ChatGPT 이미지 생성의 로그인 DAU는 얼마였나요?”라고 묻는 화면의 스크린샷으로, 메시지 아래에는 쿼리가 아직 실행 중이며 완료까지 시간이 오래 걸리고 있음을 나타내는 “1분 22초 동안 실행됨” 상태 표시가 있습니다.*
> **에이전트의 메모리는 올바른 테이블을 찾아 더 빠른 쿼리를 가능하게 합니다.**

이러한 실패를 방지하기 위해, 에이전트는 OpenAI의 데이터와 조직적 지식에 기반한 여러 계층의 맥락 위에 구축되었습니다.

> *“데이터 에이전트의 맥락 계층”이라는 제목의 다이어그램으로, 1) 테이블 사용, 2) 사람 주석, 3) Codex 보강, 4) 조직 지식, 5) 메모리, 6) 런타임 맥락의 여섯 개 계층이 쌓여 있습니다. 각 계층은 피라미드 형태의 가로 막대로 표시됩니다.*

### 계층 #1: 테이블 사용
*   **메타데이터 기반:** 에이전트는 스키마 메타데이터(열 이름과 데이터 유형)를 바탕으로 SQL 작성을 수행하며, 테이블 계보(예: 상위 및 하위 테이블 관계)를 활용해 서로 다른 테이블이 어떻게 연결되는지에 대한 맥락을 제공합니다.
*   **쿼리 추론:** 과거 쿼리를 수집함으로써 에이전트는 자체 쿼리를 작성하는 방법과 일반적으로 함께 조인되는 테이블을 이해합니다.

### 계층 #2: 사람 주석
도메인 전문가가 제공한 테이블과 열에 대한 정제된 설명으로, 스키마나 과거 쿼리만으로는 쉽게 추론하기 어려운 의도, 의미, 비즈니스적 의미, 그리고 알려진 주의 사항을 담고 있습니다. 메타데이터만으로는 충분하지 않습니다. 테이블을 확실히 구분하려면, 테이블이 어떻게 생성되었고 어디에서 비롯되었는지를 이해해야 합니다.

### 계층 #3: Codex 보강
*   테이블의 코드 수준 정의를 도출함으로써, 에이전트는 데이터에 실제로 무엇이 담겨 있는지에 대한 더 깊은 이해를 구축합니다. 
*   테이블에 저장되는 내용과 분석 이벤트로부터 어떻게 파생되는지에 대한 미묘한 차이는 추가적인 정보를 제공합니다. 예를 들어, 값의 고유성, 테이블 데이터의 업데이트 빈도, 데이터의 범위(예: 특정 필드를 제외한다면 그에 따른 세부 수준) 등에 대한 맥락을 제공할 수 있습니다.
*   이는 Spark, Python 등 다른 데이터 시스템에서 SQL을 넘어 테이블이 어떻게 사용되는지를 보여줌으로써 확장된 사용 맥락을 제공합니다.
*   이는 에이전트가 겉보기에는 비슷하지만 중요한 차이가 있는 테이블을 구분할 수 있음을 의미합니다. 예를 들어, 특정 테이블이 자사 ChatGPT 트래픽만 포함하는지 여부를 구분할 수 있습니다. 이 맥락 정보는 자동으로 갱신되므로 수동 유지 관리 없이도 최신 상태를 유지합니다.

> *“Codex로 보강된 지식 파이프라인”이라는 제목의 다이어그램입니다. 자주 사용되는 테이블이 여러 Codex 작업으로 입력되며, 이 작업들은 테이블의 목적, 데이터 단위와 기본 키, 하위 사용 패턴, 대체 테이블 옵션, 데이터 최신성 등 OpenAI 코드베이스의 세부 정보를 추출합니다.*

### 계층 #4: 조직 지식 
*   에이전트는 출시, 신뢰성 사고, 내부 코드명과 툴, 핵심 지표의 표준 정의와 계산 로직 등 중요한 회사 맥락을 담고 있는 Slack, Google Docs, Notion에 액세스할 수 있습니다.
*   이 문서들은 수집되어 임베딩 처리된 후 메타데이터와 권한 정보와 함께 저장됩니다. 검색 서비스는 런타임에 액세스 제어와 캐싱을 처리해, 에이전트가 이 정보를 효율적이고 안전하게 불러올 수 있도록 합니다.

> *12월에 커넥터 사용량이 감소한 이유를 묻는 사용자 화면의 스크린샷입니다. 에이전트는 2025년 11월 13일부터 시작된 로깅 문제로 인해 ChatGPT 5.1 출시 이후 사용량이 과소 집계되면서 감소가 발생했다고 설명합니다. 기존 텔레메트리는 더 새로운 이벤트가 기준 데이터가 될 때까지 비어 있는 상태였습니다.*

### 계층 #5: 메모리
*   에이전트가 수정 사항을 받거나 특정 데이터 질문에 대한 미묘한 차이를 발견하면, 이를 다음을 위해 저장해 사용자와 함께 지속적으로 개선할 수 있습니다. 
*   그 결과, 이후의 답변은 동일한 문제를 반복해서 겪는 대신 더 정확한 기준선에서 시작하게 됩니다.
*   메모리의 목표는 데이터 정확성에 필수적이지만 다른 계층만으로는 추론하기 어려운 비명확한 수정 사항, 필터, 제약 조건을 유지하고 재사용하는 것입니다. 
*   예를 들어, 한 사례에서 에이전트는 특정 분석 실험을 어떻게 필터링해야 하는지 알지 못했는데, 이는 실험 게이트에 정의된 특정 문자열과의 매칭에 의존했기 때문입니다. 이 경우 메모리는, 모호한 문자열 매칭을 시도하는 대신 올바르게 필터링할 수 있도록 하는 데 결정적으로 중요했습니다.
*   에이전트에 수정 사항을 제공하거나 대화에서 학습 내용을 발견하면, 다음을 위해 해당 메모리를 저장할지 묻는 안내가 표시됩니다. 메모리는 사용자가 수동으로 생성하고 편집할 수도 있습니다.
*   메모리는 전역 수준과 개인 수준으로 범위가 설정되며, 에이전트의 툴을 통해 쉽게 편집할 수 있습니다.

> *“데이터 에이전트가 메모리에 2개의 학습 내용을 저장하려고 합니다”라는 알림 배너와 함께 “ChatGPT 최상위 지표” 항목이 표시되며, 오른쪽에는 초록색 체크 표시와 함께 “전역 메모리에 저장됨”이라는 확인 메시지가 나타납니다.*

### 계층 #6: 런타임 맥락
*   테이블에 대한 사전 맥락이 없거나 기존 정보가 오래된 경우, 에이전트는 데이터 웨어하우스에 실시간 쿼리를 실행해 테이블을 직접 점검하고 조회할 수 있습니다. 이를 통해 스키마를 검증하고 데이터를 실시간으로 이해한 뒤 그에 맞게 대응할 수 있습니다.
*   에이전트는 필요에 따라 웨어하우스 외부에 존재하는 더 넓은 데이터 맥락을 얻기 위해 다른 데이터 플랫폼 시스템(메타데이터 서비스, Airflow, Spark)과도 통신할 수 있습니다.

We run a daily offline pipeline that aggregates table usage, human annotations, and Codex-derived enrichment into a single, normalized representation. This enriched context is then converted into embeddings using the OpenAI embeddings API and stored for retrieval. At query time, the agent pulls only the most relevant embedded context via retrieval-augmented generation (RAG) instead of scanning raw metadata or logs. This makes table understanding fast and scalable, even across tens of thousands of tables, while keeping runtime latency predictable and low. Runtime queries are issued to our data warehouse live as needed.

> *“데이터 에이전트의 맥락 검색”이라는 제목의 다이어그램. 오프라인 전처리 계층인 테이블 사용, 사람 주석, Codex 보강, 조직 지식, 메모리가 RAG 임베딩으로 입력됩니다. 라이브 검색에서는 에이전트가 의미 기반 검색 또는 정확한 텍스트 검색을 통해 데이터베이스를 조회해 런타임 맥락을 생성하는 모습이 표시됩니다.*

Together, these layers ensure the agent’s reasoning is grounded in OpenAI’s data, code, and institutional knowledge, dramatically reducing errors and improving answer quality.

---

## Built to think and work like a teammate
One-shot answers work when the problem is clear, but most questions aren’t. More often, arriving at the correct result requires back-and-forth refinement and some course correction.

The agent is built to behave like a teammate you can reason with. It’s a conversational, always-on and handles both quick answers and iterative exploration.

It carries over complete context across turns, so users can ask follow-up questions, adjust their intent, or change direction without restating everything. If the agent starts heading down the wrong path, users can interrupt mid-analysis and redirect it, just like working with a human collaborator who listens instead of plowing ahead.

When instructions are unclear or incomplete, the agent proactively asks clarifying questions. If no response is provided, it applies sensible defaults to make progress. For example, if a user asks about business growth with no date range specified, it may assume the last seven or 30 days. These priors allow it to stay responsive and non-blocking while still converging on the right outcome.

The result is an agent that works well both when you know exactly what you want (e.g., “Tell me about this table”) and just as strong when you’re exploring (e.g., “I’m seeing a dip here, can we break this down by customer type and timeframe?”). 

After rollout, we observed that users frequently ran the same analyses for routine repetitive work. To expedite this, the agent's workflows package recurring analyses into reusable instruction sets. Examples include workflows for weekly business reports and table validations. By encoding context and best practices once, workflows streamline repeat analyses and ensure consistent results across users.

> *“데이터 질문을 입력하세요”라는 플레이스홀더 텍스트가 있는 UI 입력 바. 그 아래에는 “워크플로 사용” 버튼이 있고, 오른쪽에는 마이크와 전송 아이콘이 있습니다. 이 바는 모서리가 둥글고 어두운 배경 위에 배치되어 있습니다.*

---

## Moving fast without breaking trust
Building an always-on, evolving agent means quality can drift just as easily as it can improve. Without a tight feedback loop, regressions are inevitable and invisible. The only way to scale capability without breaking trust is through systematic evaluation.

In this section, we’ll discuss how we leverage OpenAI’s Evals API to measure and protect the agent’s response quality.

Its Evals are built on curated sets of question-answer pairs. Each question targets an important metric or analytical pattern we care deeply about getting right, paired with a manually authored “golden” SQL query that produces the expected result. For each eval, we send the natural language question to its query-generation endpoint, execute the generated SQL, and compare the output against the result of the expected SQL.

> *“데이터 에이전트의 평가 파이프라인”이라는 제목의 다이어그램입니다. 예상 SQL이 포함된 Q&A 평가 쌍이 생성 단계로 입력되어 SQL과 결과를 생성합니다. OpenAI Evals는 데이터프레임 및 SQL 비교를 사용해 생성된 결과와 예상 결과를 비교하고 점수와 근거를 출력합니다.*

Evaluation doesn’t rely on naive string matching. Generated SQL can differ syntactically while still being correct, and result sets may include extra columns that don’t materially affect the answer. To account for this, we compare both the SQL and the resulting data, and feed these signals into OpenAI’s Evals grader. The grader produces a final score along with an explanation, capturing both correctness and acceptable variation.

These evals are like unit tests that run continuously during development to identify regressions as canaries in production; this allows us to catch issues early and confidently iterate as the agent's capabilities expand.

---

## Agent security
Our agent plugs directly into OpenAI’s existing security and access-control model. It operates purely as an interface layer, inheriting and enforcing the same permissions and guardrails that govern OpenAI’s data. 

All of the agent’s access is strictly pass-through, meaning users can only query tables they already have permission to access. When access is missing, it flags this or falls back to alternative datasets the user is authorized to use.

Finally, it's built for transparency. Like any system, it can make mistakes. It exposes its reasoning process by summarizing assumptions and execution steps alongside each answer. When queries are executed, it links directly to the underlying results, allowing users to inspect raw data and verify every step of the analysis.

---

## Lessons learned
Building our agent from scratch surfaced practical lessons about how agents behave, where they struggle, and what actually makes them reliable at scale.

### Lesson #1: Less is More
Early on, we exposed our full tool set to the agent, and quickly ran into problems with overlapping functionality. While this redundancy can be helpful for specific custom cases and is more obvious to a human when manually invoking, it’s confusing to agents. To reduce ambiguity and improve reliability, we restricted and consolidated certain tool calls.

### Lesson #2: Guide the Goal, Not the Path
We also discovered that highly prescriptive prompting degraded results. While many questions share a general analytical shape, the details vary enough that rigid instructions often pushed the agent down incorrect paths. By shifting to higher-level guidance and relying on GPT‑5’s reasoning to choose the appropriate execution path, the agent became more robust and produced better results.

### Lesson #3: Meaning Lives in Code
Schemas and query history describe a table’s shape and usage, but its true meaning lives in the code that produces it. Pipeline logic captures assumptions, freshness guarantees, and business intent that never surface in SQL or metadata. By crawling the codebase with Codex, our agent understands how datasets are actually constructed and is able to better reason about what each table actually contains. It can answer “what’s in here” and “when can I use it” far more accurately than from warehouse signals alone. 

---

## Same vision, new tools
We’re constantly working to improve our agent by increasing its ability to handle ambiguous questions, improving its reliability and accuracy with stronger validations, and integrating it more deeply into workflows. We believe it should blend naturally into how people already work, instead of functioning like a separate tool.

While our tooling will keep benefiting from underlying improvements in agent reasoning, validation, and self-correction, our team’s mission remains the same: seamlessly deliver fast, trustworthy data analysis across OpenAI’s data ecosystem.
