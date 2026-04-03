---
layout: post
title: "GPT-2로 시작하는 LLM의 원리 탐구 — 학습 리소스 총정리"
date: 2026-04-03 14:00:00 +0900
categories:
  - AI
tags:
  - llm
  - gpt-2
  - transformer
  - study-resources
---

LLM을 다루다 보면 한 번쯤 이런 생각이 들 수 있습니다.

> "Transformer의 원리를 제대로 이해하고 싶은데, 요즘 모델들은 너무 크고 복잡하다. 어디서부터 시작해야 할까?"

결론부터 말하면, **GPT-2**입니다.

GPT-2는 2019년 OpenAI가 공개한 모델로, 현대 LLM의 근간인 Transformer decoder-only 아키텍처를 가장 순수한 형태로 구현하고 있습니다. Multi-Head Attention, LayerNorm, GELU, Learned Positional Embedding — 이것이 전부입니다. RoPE도 없고, GQA도 없고, SwiGLU도 없습니다. 최신 모델들이 추가한 온갖 최적화 기법이 빠져 있기 때문에, 오히려 Transformer의 본질에 집중할 수 있습니다.

무엇보다 중요한 것은, 모델 코드와 가중치(Weight)가 전부 공개되어 있고, 이를 중심으로 한 교육 자료가 압도적으로 풍부하다는 점입니다. 성능이 아닌 **원리 파악**이 목적이라면, 지금 시점에서도 GPT-2는 가장 적합한 출발점입니다.

이 글에서는 GPT-2를 중심으로 LLM의 구조와 학습 과정을 이해하기 위한 리소스를 유튜브, 코드, 책, 아티클 순으로 정리합니다.

---

## 1. 유튜브

### Karpathy "Neural Networks: Zero to Hero" 시리즈

Andrej Karpathy(전 Tesla AI Director, OpenAI 창립 멤버)가 제작한 시리즈로, 빈 파일에서 시작하여 GPT-2를 완전히 재현하는 과정을 보여줍니다. LLM 학습 리소스 중 가장 널리 추천되는 콘텐츠입니다.

| # | 영상 | 핵심 내용 | 시간 |
|---|------|-----------|------|
| 1 | [micrograd](https://www.youtube.com/watch?v=VMj-3S1tku0) | Backpropagation과 autograd를 밑바닥부터 구현 | ~2.5h |
| 2 | [makemore 1](https://www.youtube.com/watch?v=PaCmpygFfXo) | Bigram 모델로 language modeling의 기초 이해 | ~1.5h |
| 3 | [makemore 2](https://www.youtube.com/watch?v=TCH_1BHY58I) | MLP 기반 language model 구현 | ~1.5h |
| 4 | [makemore 3](https://www.youtube.com/watch?v=P6sfmUTpUmc) | BatchNorm과 학습 과정 디버깅 기법 | ~1.5h |
| 5 | [makemore 4](https://www.youtube.com/watch?v=q8SA3rM6ckI) | Backpropagation을 수동으로 구현 | ~1.5h |
| 6 | [makemore 5](https://www.youtube.com/watch?v=t3YJ5hKiMQ0) | WaveNet 스타일 아키텍처 탐구 | ~1h |
| **7** | **[Let's build GPT](https://www.youtube.com/watch?v=kCc8FmEb1nY)** | **Transformer 전체를 처음부터 코딩** | **~2h** |
| **8** | **[Let's reproduce GPT-2](https://www.youtube.com/watch?v=l8pRSuU81PU)** | **GPT-2 (124M) 완전 재현과 학습 최적화** | **~4h** |

시리즈의 최종 목표는 7번과 8번입니다. 1~6번은 그에 필요한 기초 체력을 쌓는 과정이라고 보면 됩니다. 딥러닝 경험이 있다면 1~6은 빠르게 훑고 7-8에 집중해도 무방합니다.

### 보조 영상

- **[3Blue1Brown — Neural Networks 시리즈](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi)** — Attention 메커니즘의 시각적 직관을 쌓는 데 탁월합니다.
- **[DeepLearning.AI — How Transformer LLMs Work](https://www.deeplearning.ai/short-courses/how-transformer-llms-work/)** — Jay Alammar와 Andrew Ng의 공동 제작 무료 과정(~90분)으로, 애니메이션을 활용해 Transformer 아키텍처의 각 구성요소를 설명합니다.

---

## 2. 코드

### 핵심 저장소

**[karpathy/build-nanogpt](https://github.com/karpathy/build-nanogpt)**

위 유튜브 8번 영상의 코드 저장소입니다. 빈 파일에서 출발하여 GPT-2 (124M)을 완성해 나갑니다. 이 저장소의 특징은 **git commit이 학습 단계별로 정리되어 있다**는 점입니다. `git log`를 따라가며 각 커밋의 변경 사항을 읽으면, 왜 그 코드가 필요한지 자연스럽게 이해할 수 있습니다. 원리 학습 목적이라면 가장 먼저 살펴봐야 할 저장소입니다.

**[karpathy/nanoGPT](https://github.com/karpathy/nanoGPT)**

완성형 학습/파인튜닝 코드입니다. `train.py` 약 300줄, `model.py` 약 300줄로 구성되어 있으며, 이보다 간결한 GPT 구현은 찾기 어렵습니다. 코드를 이해한 후 실제로 학습을 돌려보는 단계에서 사용하기 좋습니다.

참고로 nanoGPT는 2025년 11월부로 deprecated 되었습니다. Karpathy가 후속 프로젝트인 [nanochat](https://github.com/karpathy/nanochat)을 공식 발표했고, nanoGPT README에도 "nanochat을 대신 사용하라"고 명시되어 있습니다. nanochat은 pretraining부터 SFT(Supervised Fine-Tuning), 추론, 웹 UI까지의 풀스택 파이프라인을 약 8,000줄로 구현한 프로젝트입니다. 다만 **원리 학습이 목적이라면 nanoGPT가 여전히 더 적합합니다.** 코드가 훨씬 짧고 단순하기 때문입니다. nanochat은 기본 원리를 충분히 이해한 후 실전으로 넘어가는 단계에서 참고하면 됩니다.

**[karpathy/llm.c](https://github.com/karpathy/llm.c)**

C/CUDA로 GPT-2 학습을 구현한 프로젝트입니다. PyTorch 없이 forward pass와 backward pass를 밑바닥에서 작성합니다. "Transformer가 실제로 GPU 위에서 어떻게 동작하는가"라는 질문에 대한 가장 직접적인 답이 이 저장소에 있습니다. 현재 Karpathy가 가장 활발히 업데이트하고 있는 프로젝트이기도 합니다. GEMM(General Matrix Multiply)이 GPU 성능을 어떻게 끌어내는지 코드를 훑어보는 것만으로도 상당한 엔지니어링 인사이트를 얻을 수 있습니다.

**[karpathy/minbpe](https://github.com/karpathy/minbpe)**

BPE(Byte Pair Encoding) 토크나이저를 밑바닥부터 구현한 프로젝트입니다. LLM의 입력 처리가 실제로 어떻게 동작하는지, 왜 한국어 처리 시 토큰 효율이 급격히 떨어지는지, 왜 LLM이 간단한 문자열 역순 출력조차 어려워하는지 — 이러한 의문들의 답이 Tokenization에 있습니다. Karpathy의 [Tokenization 영상](https://www.youtube.com/watch?v=zduSFxRajkE)과 함께 보면 이해가 한층 깊어집니다. 한국어 텍스트를 직접 넣어서 tiktoken의 결과와 비교해보는 실험도 권합니다.

**[rasbt/LLMs-from-scratch](https://github.com/rasbt/LLMs-from-scratch)**

아래 책 섹션에서 소개할 Sebastian Raschka 저서의 전체 코드 저장소입니다. Jupyter notebook 형태로 제공되어, 각 단계를 실행하면서 따라가기에 적합합니다.

### 실습 순서 제안

코드 저장소를 어떤 순서로 접근하면 좋을지 정리하면 다음과 같습니다.

```
1. build-nanogpt의 커밋을 하나씩 따라가며 코드의 의도를 이해한다
2. nanoGPT로 Shakespeare char-level 학습을 돌려본다 (수 분 소요)
3. nanoGPT로 OpenWebText에서 GPT-2 124M 재현을 시도한다
4. 자체 데이터셋으로 파인튜닝 실험을 진행한다
```

---

## 3. 책

### Build a Large Language Model (From Scratch)

**저자**: Sebastian Raschka (2024, Manning)
**코드**: [rasbt/LLMs-from-scratch](https://github.com/rasbt/LLMs-from-scratch)

Feynman의 "만들 수 없으면 이해한 것이 아니다"를 그대로 실천하는 책입니다. HuggingFace Transformers와 같은 고수준 라이브러리에 의존하지 않고, 순수 PyTorch만으로 GPT-2 스타일 LLM을 구축합니다.

Tokenization과 Embedding에서 출발하여, Attention 메커니즘 구현, GPT 아키텍처 설계, Pretraining 파이프라인 구축, 텍스트 분류 및 Instruction Following을 위한 Fine-tuning, 그리고 LoRA를 활용한 Parameter-Efficient Fine-Tuning까지를 다룹니다.

Karpathy의 영상이 "라이브 코딩을 따라가며 전체 흐름을 체감하는 것"이라면, 이 책은 "각 구성요소를 체계적으로 정리하며 빈틈을 메우는 것"에 해당합니다. 둘을 병행하면 상호 보완적인 학습 효과를 얻을 수 있습니다.

### 보조 도서

| 책 | 설명 |
|----|------|
| **Build a Reasoning Model (From Scratch)** — Raschka, 2025 | 위 책의 후속편입니다. Pretrained LLM에 추론(Reasoning) 능력을 추가하는 과정을 다루며, DeepSeek R1이나 OpenAI o1 등에서 활용되는 Inference-time Scaling, 강화학습(RL), Distillation 기법을 설명합니다 |
| **Hands-On Large Language Models** — Jay Alammar & Maarten Grootendorst | Illustrated Transformer 시리즈의 확장판으로, 시각적 설명이 탁월합니다. 4장부터 텍스트 분류, Semantic Search 등의 실습이 포함되어 있습니다 |

---

## 4. 아티클 및 기타 리소스

### 필수 아티클

**[The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)** — Jay Alammar

Transformer 시각화의 정석으로, 2018년에 작성되었지만 지금까지도 가장 먼저 추천되는 글입니다. Self-attention이 무엇인지, Encoder-Decoder가 어떻게 연결되는지를 직관적인 도식 하나로 파악할 수 있습니다.

**[The Illustrated GPT-2](https://jalammar.github.io/illustrated-gpt2/)** — Jay Alammar

위 글의 GPT-2 특화 버전입니다. Decoder-only 아키텍처가 어떻게 텍스트를 생성하는지 시각적으로 보여줍니다.

**[Attention Is All You Need](https://arxiv.org/abs/1706.03762)** — Vaswani et al., 2017

Transformer 원논문입니다. 한 번은 읽어야 합니다. Illustrated Transformer를 먼저 읽고 나면 훨씬 수월하게 따라갈 수 있습니다.

**[Language Models are Unsupervised Multitask Learners](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf)** — Radford et al., 2019

GPT-2 원논문입니다. 모델 아키텍처의 세부 사항보다는, "Unsupervised Pretraining만으로도 다양한 하위 태스크를 수행할 수 있다"는 핵심 아이디어를 파악하는 데 의의가 있습니다.

### 대학 강의 (무료 공개)

- **[Stanford CS336: Language Modeling from Scratch](https://cs336.stanford.edu/)** — LLM을 처음부터 구축하는 고급 과정으로, 과제 코드가 공개되어 있습니다.
- **[Stanford CS224N: NLP with Deep Learning](https://web.stanford.edu/class/cs224n/)** — Transformer와 Attention 이론의 기초를 다지는 강의입니다.

### 블로그 및 뉴스레터

- **[Lilian Weng — The Transformer Family v2](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family-v2/)** — Transformer 아키텍처의 다양한 변형을 포괄적으로 정리한 글입니다.
- **[Sebastian Raschka's Substack](https://magazine.sebastianraschka.com/)** — LLM 연구 동향과 Fine-tuning 실험 결과를 꾸준히 공유합니다.
- **[Cameron Wolfe — Deep Learning Focus](https://cameronrwolfe.substack.com/)** — Decoder 아키텍처, SFT, RLHF 등에 대한 깊이 있는 해설을 제공합니다.

---

## 추천 학습 경로

마지막으로, 위 리소스들을 어떤 순서로 활용하면 좋을지 정리합니다.

**Step 1.** Jay Alammar의 Illustrated Transformer / GPT-2를 읽습니다.
Transformer의 전체 구조를 시각적으로 파악하는 단계입니다.

**Step 2.** Karpathy의 영상 7-8번을 따라 코딩합니다.
직접 코드를 작성하면서 "실제로 동작하는 과정"을 체감하는 단계입니다.

**Step 3.** Raschka의 책으로 체계적으로 정리합니다.
영상에서 빠르게 지나간 개념들을 꼼꼼히 채우고, Fine-tuning까지 확장하는 단계입니다.

**Step 4.** nanoGPT로 직접 실험합니다.
Shakespeare 데이터 → OpenWebText → 자체 데이터셋 순으로 범위를 넓혀가면 됩니다.

**Step 5. (선택)** 더 깊이 파고 싶다면 다음을 권합니다.
- minbpe로 토크나이저를 직접 만들어보기 (한국어 토큰 효율 실험 포함)
- llm.c로 C/CUDA 레벨에서의 학습 과정을 이해하기
- nanochat으로 Pretraining부터 서빙까지의 풀스택 파이프라인을 경험하기

---

성능이 뛰어난 모델을 만드는 것은 그다음 문제입니다.
먼저 124M 파라미터의 작은 모델이 어떻게 텍스트를 생성하는지, 그 과정을 코드 한 줄 한 줄 따라가며 이해하는 것이 이 여정의 본래 목표입니다.
