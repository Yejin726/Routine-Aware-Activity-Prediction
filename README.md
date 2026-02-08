# Routine-Aware-Activity-Prediction

Routing prediction 성능은 사용되는 추론 매커니즘에 따라 단계적으로 향상된다.

- H1. Prompt-only baseline
  LLM은 단일 프롬프트 기반 추론만으로 기본적인 반복 루틴을 학습할 수 있다.
  입력: 현재 시각, 요일, 최근 행동 로그(6개)
  제약:  과거 사례를 명시적으로 참조하지 않음
        숨겨진 규칙(주기성, 외생 요인)을 명시적으로 추론하지 않음
  기대:  강하게 반복되는 루틴에서 일정 수준의 성능
  
- H2. RAG-only
  RAG를 이용하여 과거 유사 사례를 참조하면 Routine prediction 성능이 향상된다
  추가요소 :
    과거 life-logging 데이터 검색
    유사한 날짜/행동 패턴을 참고
  기대:
    단순 반복 루틴 + 약한 변동성 환경에서 성능 향상
    Retrieval noise(요일 혼합, 문서 단위 문제)에 취약
  
- H3. Agentic(multi-stage reasoning)
  Agentic 구조는 관찰 불완전성과 숨겨진 규칙이 존재하는 환경에서 추가적인 성능 향상을 제공한다.
  Agentic 구성 요소:
    상태 해석 (context understanding)
    belief inference(날씨, 예외 발생 여부 등)
    도구 선택 (RAG 사용 여부 결정)
    최종 의사 결정
  기대 :
    불확실성 또는 분기가 존재하는 regime에서 효과적


<img width="613" height="374" alt="image" src="https://github.com/user-attachments/assets/e9953e27-5101-431d-a970-ad0b172bd248" />


<img width="691" height="374" alt="image" src="https://github.com/user-attachments/assets/b2614fd0-ab83-41c5-93db-2311869e29e8" />


<img width="442" height="178" alt="image" src="https://github.com/user-attachments/assets/eaf6c566-b3c2-4214-99a0-90b6fb15f4e4" />
