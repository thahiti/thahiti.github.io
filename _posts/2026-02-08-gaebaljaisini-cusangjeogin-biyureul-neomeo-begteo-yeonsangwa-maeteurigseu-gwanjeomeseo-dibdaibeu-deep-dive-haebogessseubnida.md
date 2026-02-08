---
layout: post
title: "개발자이시니, 추상적인 비유를 넘어 벡터 연산과 매트릭스 관점에서 딥다이브(Deep Dive) 해보겠습니다."
date: 2026-02-08 22:36:09 +0900
categories: [DiscordLog]
tags: [til, dev]
---

개발자이시니, 추상적인 비유를 넘어 벡터 연산과 매트릭스 관점에서 딥다이브(Deep Dive) 해보겠습니다.
셀프 어텐션(Self-Attention)의 핵심은 **"입력된 각 토큰(단어)을 문맥을 반영한 새로운 벡터로 변환하는 과정"**입니다. 이 과정은 학습 가능한 가중치 행렬(W^Q, W^K, W^V)을 통해 이루어집니다.
1. 입력 (Input)과 가중치 행렬 (Weight Matrices)
문장의 각 단어는 고정된 크기의 임베딩 벡터(x)로 변환되어 들어옵니다. 셀프 어텐션 레이어에는 3개의 학습 가능한 가중치 행렬이 존재합니다.
• W^Q (Query Weights)
• W^K (Key Weights)
• W^V (Value Weights)
입력 벡터 x에 이 행렬들을 곱해 3가지 새로운 벡터를 만들어냅니다.
