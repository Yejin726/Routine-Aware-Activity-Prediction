# Routine-Aware-Activity-Prediction


## 가설
Routing prediction 성능은 사용되는 추론 매커니즘에 따라 단계적으로 향상된다.


- H1. Prompt-only baseline

  가설: LLM은 단일 프롬프트 기반 추론만으로 기본적인 반복 루틴을 학습할 수 있다.  
  입력: 현재 시각, 요일, 최근 행동 로그(6개)  
  제약:  과거 사례를 명시적으로 참조하지 않음. 숨겨진 규칙(주기성, 외생 요인)을 명시적으로 추론하지 않음        
  기대:  강하게 반복되는 루틴에서 일정 수준의 성능

  
- H2. RAG-only

  가설: RAG를 이용하여 과거 유사 사례를 참조하면 Routine prediction 성능이 향상된다  
  추가요소 :  
    과거 life-logging 데이터 검색  
    유사한 날짜/행동 패턴을 참고  
  기대:  
    단순 반복 루틴 + 약한 변동성 환경에서 성능 향상  
    Retrieval noise(요일 혼합, 문서 단위 문제)에 취약  
    
  
- H3. Agentic(multi-stage reasoning)

  가설: Agentic 구조는 관찰 불완전성과 숨겨진 규칙이 존재하는 환경에서 추가적인 성능 향상을 제공한다.  
  Agentic 구성 요소:  
    상태 해석 (context understanding)  
    belief inference(날씨, 예외 발생 여부 등)  
    도구 선택 (RAG 사용 여부 결정)  
    최종 의사 결정  
    
  기대 :  불확실성 또는 분기가 존재하는 regime에서 효과적
    



## Synthetic Life-Logging Dataset 개요

  본 연구는 실제 사용자 로그 대신,  
  명시적 루틴 규칙 + 확률적 변동성을 결합한 synthetic life-logging dataset을 사용  
  해당 데이타셋은 규칙적인 루틴에 불규칙성을 부여하여 숨겨진 규칙을 추론 할 수 있는지 실험하였음.  
  

  ### 기본 설정:  
    시간 단위 : 30분  
    하루 슬롯 : 48  
    
  ### 기본 루틴 정의  
    - Weekday regime (Mon-Fri)  
      06:00 기상 → 출근 → 업무 → 점심 → 업무 → 퇴근  
      저녁: 요일별 고정 이벤트  
        Mon/Wed/Fri: 영어 학원  
        Tue/Thu: 운동  
        
    - Saturday regime   
      10:00 기상 → 오전 휴식 → 오후 게임 → 저녁 외출 →  22:00 취침  
      숨겨진 규칙:  
        홀수 주: 친구 만남  
        짝수 주: 가족 만남  
        
    - Sunday regime  
      08:00 기상 → 교회 일정 → 점심 → 오후 운동 또는 휴식 → 저녁 게임 → 22:00 취침  
      
    - 부여된 불규칙성  
      비가 오면(20% 발생):   
        운동 취소 → 휴식으로 변경,  
        지하철 → 택시  
      늦잠 발생(10% 발생):  
        지하철 → 택시  

  ### 데이타셋 예시
 <img width="549" height="649" alt="image" src="https://github.com/user-attachments/assets/63cf8965-7083-4bc7-b94c-6c6acf5ace03" />
 

##실험 결과

 
### Prompt-only Baseline 
- Prompt  
```text
You are predicting the user's next 30-minute activity.

Calendar context:
- day_of_week: Saturday
- week_index: 1

Recent activity history:
10:00 wake_up
10:30 breakfast
11:00 relax_screen

What is the next activity?

Output ONE label only.
```
- 성능
```text
Topk-1 accuracy = 57.92%
```  
  과거 사례 참조 없으며 숨겨진 규칙(주기/날씨) 추론을 단독 수행해야 함  

### RAG-only (Retrieval-Augmented)  
- prompt
```text
You are predicting the user's next 30-minute activity from a life-log.

Calendar context:
- day_of_week: {day_of_week}
- week_index: {week_index}

Recent activity history:
{history_lines}

Retrieved past examples:
{retrieved_examples_text}

Question:
What is the most likely next activity at {next_ts}?

Allowed labels:
{label_list}

Rules:
- Use retrieved examples if helpful.
- Output exactly ONE label from the allowed labels.
- Output the label only.
```
- 성능
```text
Topk-1 accuracy = 65%
```  
  prompt-only 대비 "패턴 매칭" 성능 상승

### Agentic_v1
 Agentic 설계 초기 버전. 발생할 수 있는 불확실성에 대한 가설을 세우고 RAG에 반영함  
 
 #### 구조  
   - Observable Context  
      모델이 실제로 관측 가능한 정보만 제공하는 입력 영역  
      
   - Belief Inference Agent  
     관측 불가능한 latent factor에 대한 가설을 세움  
     예시 belif :  
       비가 왔을 가능성  
       늦잠 여부  
       오늘 루틴이 예외 상황인지    
     
   - RAG  
     현재 belief와 유사한 과거 사럐를 검색해 증거 제공  
       
   - Decision / Planning Agent  
     모든 정보를 종합해 최종 다음 행동 1개를 예측  

 ```
┌──────────────────────┐
│ Observable Context   │
│ (Given to LLM)       │
│                      │
│ - day_of_week        │
│ - week_index         │
│ - recent activities  │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ Belief Inference     │
│ Agent (Always ON)    │
│                      │
│ - rain?              │
│ - late wakeup?       │
│ - regime guess       │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ RAG (Always or       │
│ Frequently Used)     │
│                      │
│ Retrieve past cases  │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ Decision / Planning  │
│ Agent                │
│                      │
│ Predict next action  │
└──────────────────────┘
```
- 성능
```text
Topk-1 accuracy = 58.33%
```
- 분석  
  왜 되려 성능이 떨어졌는가?
  
   - RAG는 생각 하지 않기에 과거 유사 사례를 패턴 매칭  
   - weekday의 강한 반복성, 토요일의 고정 시간 구조와 같은 low-entropy routine 구조에서 매우 강함  
   - 기존 RAG에 Belief inference를 추가 하면서 불규칙성에 대한 추론을 모든 경우에 수행한다(Overthinking)  
   - life-log의 대부분의 경우 규칙적인 구조이기 때문에 모든 timestep에 대해 추론할 필요가 없음  
   - 모든 경우에 수행된 Belief error가 RAG에 영향을 미치고 더 나아가 결과에 영향을 끼쳤을 것으로 예상됨  
   - 결론적으론 불필요한 추론이 반복됨
 

### Agentic V2  
규칙적인 라이프 루틴에 희소하게 일어나는 불규칙성을 찾기 위해 모든 timestep마다 이 불규칙성을 찾는 것은 매우 비효율 적이다.  
--> 불규칙이 일어나는 분기에만 reasoning을 하자

 #### 구조  
   - Entropy / Uncertainty Estimator
    지금 예측 문제가 추론이 필요한지 여부를 결정하는 게이트

   - Context Inspector
     현재 상황이 "평범한 반복 루틴"인지 빠르게 스캔

```
┌──────────────────────┐
│ Observable Context   │
│ (Given to LLM)       │
│                      │
│ - day_of_week        │
│ - week_index         │
│ - recent activities  │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ Context Inspector    │
│ (Lightweight)        │
│                      │
│ - time / day check   │
│ - pattern stability  │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ Entropy / Uncertainty│
│ Estimator            │
│                      │
│ Is this deterministic│
│ or branching?        │
└───────┬───────┬──────┘
        │       │
   low-entropy  high-entropy
        │       │
        ▼       ▼
┌────────────┐  ┌──────────────────────┐
│ Direct     │  │ Belief Inference     │
│ Prediction │  │ Agent                │
│ (No agent) │  │                      │
└──────┬─────┘  │ - rain?              │
       │        │ - late wakeup?       │
       │        │ - regime type        │
       │        └─────────┬────────────┘
       │                  │
       │                  ▼
       │        ┌──────────────────────┐
       │        │ RAG (Optional)        │
       │        │                      │
       │        │ Retrieve past cases  │
       │        └─────────┬────────────┘
       │                  │
       └──────────┬───────┘
                  ▼
        ┌──────────────────────┐
        │ Decision / Planning  │
        │ Agent                │
        │                      │
        │ Predict next action  │
        └──────────────────────┘
```

- 성능
```text
Topk-1 accuracy = 67.92%
```

- 분석  
  - v1의 문제였던 모든 timestep에서 동일한 추론 파이프라인을 실행하면서 불필요한 belief inference를 수행  
  - 결과가 RAG에 비해 크게 떨어지는 이유는 대부분의 데이타가 RAG에 적절한 low-entroy로 구성되어 있기 때문
  - 그렇기에 Belief Inference를 수행하기 전 현재 상황의 entroy를 판단하고 불확실성이 없는 구간에선 추론을 하지 않고 단순 예측을 수행  
  - 불확실성이 예상되는 구간에선 해당 정보를 RAG로 불러와 교차 검증을 수행하고 결과를 결정하게 함으로서 성능 개선 
     

   

## 실험 결과
<img width="613" height="374" alt="image" src="https://github.com/user-attachments/assets/e9953e27-5101-431d-a970-ad0b172bd248" />


<img width="691" height="374" alt="image" src="https://github.com/user-attachments/assets/b2614fd0-ab83-41c5-93db-2311869e29e8" />


<img width="442" height="178" alt="image" src="https://github.com/user-attachments/assets/eaf6c566-b3c2-4214-99a0-90b6fb15f4e4" />
