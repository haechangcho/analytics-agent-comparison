# analytics-agent-comparison

OpenAI · Meta · Anthropic 세 회사가 2026년 상반기에 공개한 **사내 데이터 분석 에이전트 구축 사례**의 원문과 비교 분석을 모은 저장소입니다. 클론해서 Claude Code 등으로 질의응답하는 용도로 만들었습니다.

## 무엇인가

세 회사가 비슷한 시기에 "자연어 질문 → 정확한 분석"이라는 같은 문제를 어떻게 풀었는지 글로 공개했습니다. 각 블로그 원문을 그대로 크롤링해 두고, 차이점과 공통점을 기준으로 접근법을 정리했습니다.

## 파일 구성

| 파일 | 내용 |
|---|---|
| [`comparison.md`](comparison.md) | **세 회사 비교 분석** (먼저 읽기) — 접근법 / 공통점 / 차이점 |
| [`openai.md`](openai.md) | OpenAI 원문 (6-Layer Context Stack) |
| [`meta.md`](meta.md) | Meta 원문 (Cookbook / Recipe / Ingredient) |
| [`claude.md`](claude.md) | Anthropic 원문 (3 Failure Modes → 4-Layer Stack) |

## 어떻게 쓰나

```bash
git clone https://github.com/haechangcho/analytics-agent-comparison.git
cd analytics-agent-comparison
claude   # Claude Code 실행 후 질문
```

원문 전체가 repo 안에 있으므로, 클론한 디렉터리에서 바로 물어볼 수 있습니다. 예시 질문:

- "세 회사가 '정답의 출처'를 각각 무엇으로 보는지 한 문장으로 요약해줘"
- "우리 팀 규모(데이터 소수, 비기술 사용자 多)에는 어떤 접근이 맞을까?"
- "Anthropic이 직접 '효과가 없었다'고 기록한 실험들만 정리해줘"
- "Meta의 Recipe와 Anthropic의 Skill의 차이가 뭐야?"

## 출처

- Anthropic: https://claude.com/blog/how-anthropic-enables-self-service-data-analytics-with-claude
- OpenAI: https://openai.com/ko-KR/index/inside-our-in-house-data-agent/
- Meta: https://medium.com/@AnalyticsAtMeta/inside-metas-home-grown-ai-analytics-agent-4ea6779acfb3
