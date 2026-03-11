---
layout: post
title: "1. Pure Python (NumPy / scikit-learn)"
date: 2026-03-11 21:57:10 +0900
categories: [DiscordLog]
tags: [til, dev]
---

1. Pure Python (NumPy / scikit-learn)
가장 가벼운 방식으로, vector search는 본질적으로 행렬 곱셈이며 Python에는 이미 그에 최적화된 도구가 있다 는 점을 활용합니다.
    ∙    sentence-transformers로 embedding 생성 → NumPy 배열로 저장 → dot product로 검색
    ∙    100만 벡터 × 384차원(float32) 기준 약 1.5GB RAM이면 충분 하므로 상당한 규모까지 커버 가능
    ∙    np.save/pickle로 전처리 결과를 디스크에 저장해두고, 서비스 시작 시 메모리에 로드
    ∙    장점: 의존성 최소, 디버깅 용이, 네트워크 레이턴시 없음
    ∙    단점: metadata filtering이 불편하고 CRUD 불가 (immutable array)
2. FAISS (Facebook AI Similarity Search)
가장 대중적인 in-memory vector search 라이브러리입니다. 서버리스로 동작하며 프로세스 내에서 직접 실행되고, 별도 인프라 없이 메모리에 벡터를 저장 합니다.
    ∙    IndexFlatL2: 정확한 brute-force 검색 (소규모에 적합)
    ∙    IndexIVFFlat: 클러스터링 기반 근사 검색 (중규모)
    ∙    IndexHNSW: 그래프 기반 근사 검색 (속도 우선)
    ∙    faiss.write_index() / faiss.read_index()로 전처리된 인덱스를 파일로 persist
    ∙    LangChain과의 통합이 잘 되어 있어 RAG 파이프라인 구축이 편리
3. Embedded Vector DB (ChromaDB / LanceDB / sqlite-vec)
별도 서버 프로세스 없이 애플리케이션에 내장되는 DB들입니다.
ChromaDB
    ∙    chromadb.EphemeralClient()로 in-memory 모드 사용 가능하며 프로그램 종료 시 데이터 소멸
    ∙    vector search, full-text search, metadata filtering, hybrid search(BM25 + dense vector)를 모두 지원
    ∙    내부적으로 hnswlib + SQLite 사용
LanceDB
    ∙    서버 없이 동작하며 자체 Lance 파일 포맷으로 벡터를 저장하는 serverless 구조
    ∙    Rust로 구현되어 리소스 사용이 효율적
    ∙    디스크 기반이지만 memory-mapped I/O로 빠름
sqlite-vec
    ∙    SQLite 확장으로 별도 인프라 없이 완전히 self-contained RAG 파이프라인을 구현  가능
    ∙    기존 SQLite 에코시스템 활용 가능
추천 아키텍처
Randy의 조건(제한 규모, 업데이트 없음, 별도 인프라 없음)에 가장 적합한 구성:

[전처리 단계]
문서 → chunking → sentence-transformers embedding → FAISS index 저장 (or pickle)

[서비스 단계]
앱 시작 → FAISS index 메모리 로드 → query embedding → similarity search → LLM에 context 주입


규모별 추천:
    ∙    ~1만 chunks 이하: NumPy 순수 구현이면 충분
    ∙    ~10만 chunks: FAISS IndexFlatL2 (정확 검색도 수 ms)
    ∙    ~100만 chunks: FAISS IndexIVFFlat 또는 HNSW
    ∙    metadata filtering 필요: ChromaDB in-memory 모드
FAISS가 가장 현실적인 선택입니다. 전처리 후 인덱스 파일만 저장해두면 서비스 시작 시 로드만 하면 되고, 별도 DB 서버가 필요 없습니다.​​​​​​​​​​​​​​​​
