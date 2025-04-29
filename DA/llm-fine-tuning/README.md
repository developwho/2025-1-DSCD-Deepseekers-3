이 모델은 구글의 genai에서 제공하는 파인튜닝 전용 모델 'gemini-1.5-flash-001-tuning'을 사용합니다. 후속 모델도 가능은 하나 전용 함수 활용 및 안정성을 위해 해당 모델을 사용합니다. 

## 📄 목차

1.  [이 노트북은 무엇을 하나요?](#1-이-노트북은-무엇을-하나요)
2.  [필요한 것들](#2-필요한-것들)
3.  [시작하기](#3-시작하기)
4.  [노트북 실행 단계별 설명](#4-노트북-실행-단계별-설명)
5.  [데이터 형식 (학습 및 출력)](#5-데이터-형식-학습-및-출력)
---

## 1. 이 노트북은 무엇을 하나요?

일단 **YouTube 댓글 형태의 한국어 편의점 디저트 리뷰**는 정해진 형식이 없습니다. 하지만 분석이나 통계를 내려면 **맛, 식감, 가격, 재구매 의사** 등 중요한 정보들을 구조화해야 합니다.

이를 위해 **Gemini**를 우리의 특정 작업(리뷰에서 JSON 추출)에 맞게 **Fine-tuning (미세 조정)** 하는 과정을 진행합니다. Fine-tuning을 통해 모델은 특정 패턴(리뷰 텍스트 → 특정 JSON 형식)을 학습하여, 나중에는 새로운 리뷰가 들어와도 자동으로 별도의 shot 없이 JSON 형식의 데이터 분석 결과를 만들어낼 수 있게 됩니다.

또한, 모델이 완벽하게 JSON만 출력하지 않는 경우를 대비해, 출력 결과에서 유효한 JSON 부분만 **깔끔하게 추출**하여 데이터 분석에 바로 사용할 수 있는 형태로 **CSV 파일로 저장**하는 방법까지 포함하고 있습니다.

## 2. 필요한 것들

*   **Google 계정:** Colab 노트북을 사용하고 Google Cloud 리소스에 접근하기 위해 필요합니다.
*   **Google Cloud Project:** GCS (Google Cloud Storage) 버킷을 만들기 위해 필요합니다. (프로젝트 ID가 필요합니다.)
*   **Google Cloud Storage (GCS) 버킷:** Fine-tuning 학습 데이터를 저장할 공간입니다. (버킷 이름이 필요합니다.)
*   **Google Generative AI API 키:** Gemini API에 접근하기 위해 필요하며, **Fine-tuning 권한**과 **GCS 버킷 읽기 권한**이 있어야 합니다.
*   **학습 데이터 파일:** 위에서 설명한 **JSON Lines (JSONL)** 형식의 파일이 GCS 버킷에 업로드되어 있어야 합니다.

## 3. 시작하기

1.  이 `.ipynb` 파일을 다운로드 받거나 Google Drive에 복사하여 **Google Colab**에서 엽니다.
2.  Colab 노트북 메뉴에서 `Add-ons` (추가 기능) > `Secrets` (비밀)로 이동합니다.
3.  `New secret` (새 비밀)을 클릭하고, `Name`에 `GOOGLE_API_KEY`, `Value`에 발급받은 Google AI Studio API 키를 붙여넣고 저장합니다. 이 노트북에서 `GOOGLE_API_KEY`를 사용할 수 있도록 스위치를 켭니다.
4.  노트북의 각 코드 셀을 순서대로 실행하면서, **`--- USER INPUT REQUIRED ---`** 표시가 있는 부분에는 사용자의 GCP 프로젝트 ID, GCS 버킷 이름, JSONL 파일 경로 등을 정확하게 입력합니다.

## 4. 노트북 실행 단계별 설명

노트북은 다음과 같은 단계로 구성되어 있습니다. 각 셀을 순서대로 실행하세요.

*   **Step 1-4: 환경 설정 및 API 클라이언트 초기화**
    *   필요한 Python 라이브러리들을 설치합니다.
    *   Colab Secrets에 저장한 API 키를 불러옵니다.
    *   Gemini API 클라이언트를 초기화하고, Fine-tuning이 가능한 모델 목록을 확인하여 설정이 올바른지 검증합니다.
*   **Phase 3.5: 학습 데이터 검증 (중요!)**
    *   **이 섹션은 학습 데이터를 GCS에 올리기 전에, 또는 GCS에서 다운받아 확인하기 위해 사용될 수 있습니다.** 여기서는 JSONL 파일의 각 라인이 유효한 JSON 형식인지, 그리고 우리가 정의한 **JSON 스키마를 제대로 따르는지** 자동으로 검사합니다.
    *   스키마에 맞지 않는 데이터는 Fine-tuning에 사용하면 오히려 모델 성능을 해치므로, 여기서 걸러내거나 수정해야 합니다.
    *   검증을 통과한 데이터는 `finetuning_data_validated.jsonl` 파일로 저장됩니다.
*   **Step 5 & 6: GCS 설정 및 학습 데이터 준비**
    *   GCP 프로젝트 ID, GCS 버킷 이름, 검증된 JSONL 파일의 GCS 경로(`gs://...`)를 입력합니다.
    *   **중요:** 현재 사용 중인 Gemini API (Developer API) 방식은 GCS에서 직접 Fine-tuning을 지원하지 않으므로, 이 단계에서 GCS의 JSONL 파일을 Colab 환경으로 **다운로드**하여 메모리에 로드합니다.
    *   Fine-tuning에 사용할 기본 모델(`gemini-1.5-flash-001-tuning` 등)과 에포크 수, 배치 크기 등 학습 파라미터를 설정합니다.
*   **Step 7: Fine-tuning 작업 시작 및 모니터링**
    *   설정한 데이터와 파라미터를 사용하여 Fine-tuning 작업을 시작합니다.
    *   이 작업은 Google의 백엔드 서버에서 실행되며, 완료될 때까지 시간이 소요될 수 있습니다 (데이터 양과 서버 부하에 따라 몇 분~몇 시간).
    *   코드 셀은 작업 상태를 주기적으로 보여주며 완료될 때까지 기다립니다.
    *   **작업이 성공하면, 튜닝된 모델의 고유한 이름(예: `tunedModels/your-model-name-identifier`)이 출력됩니다. 이 이름이 다음 단계에서 필요합니다.**
*   **Step 8: 튜닝된 모델 테스트**
    *   Step 7에서 얻은 튜닝된 모델 이름을 사용하여 새로운 리뷰 텍스트들에 대한 결과를 생성해 봅니다.
    *   모델의 원시 출력과, 그 출력이 유효한 JSON인지 검증하는 과정을 보여줍니다. 모델이 JSON 앞에 불필요한 텍스트를 추가하는 경우, 코드의 후처리 로직이 작동하여 JSON 부분만 추출하는 것도 확인합니다.

## 5. 데이터 형식 (학습 및 출력)

**학습 데이터 (GCS JSONL 파일):**

각 라인은 JSON 객체이며, 반드시 두 개의 키를 가져야 합니다.

```json
{"text_input": "원본 리뷰 텍스트 (문자열)", "output": "모델이 생성해야 할 목표 JSON 객체를 문자열로 변환한 것"}
```

**모델 출력 (목표 JSON 스키마):**

모델이 학습을 통해 생성하도록 목표한 JSON 구조입니다.

```json
{
  "type": "object",
  "properties": {
    "attributes": { /* ... 속성 상세 스키마 ... */ },
    "meta": { /* ... 메타 정보 상세 스키마 ... */ },
    "is_noise": { "type": "boolean" },
    "overall_sentiment": { "type": ["string", "null"], "enum": ["positive", "negative", "mixed", "neutral", null] }
  },
  "required": ["attributes", "meta", "is_noise", "overall_sentiment"]
}
```
(전체 상세 스키마는 노트북 Step 6의 `TARGET_JSON_SCHEMA` 변수를 참고하세요.)
