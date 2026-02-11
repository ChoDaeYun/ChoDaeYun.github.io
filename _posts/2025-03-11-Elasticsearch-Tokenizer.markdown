---
layout: post
title:  "Elasticsearch 형태소 분석을 통한 검색 최적화"
description: 형태소 분석기를 활용한 한국어 검색 문제 해결
date:   2025-03-11 00:00:00 +000
categories: Elasticsearch Search Tokenizer
---

## 개요
Elasticsearch에서 한국어 검색 시 발생하는 문제와 형태소 분석기를 통한 해결 방법

## 문제 상황

검색 기능 구현 중 다음과 같은 문제가 발생했습니다.

**예시:**
- 제목: "철수가 개발을 하고 있습니다."
- 검색어: "철수"
- 결과: **검색되지 않음**

### 원인 분석
Elasticsearch에서 검색이 되지 않는 근본적인 이유는 **품사(Part of Speech) 불일치** 때문입니다.

**문제의 핵심:**
- 저장된 데이터: "철수가" (명사 + 조사)
- 검색어: "철수" (명사)
- 결과: 단어 자체는 포함되어 있지만, 품사 형태가 다르기 때문에 매칭되지 않음

Elasticsearch의 기본 분석기는 한국어 조사를 분리하지 못하고 "철수가"를 하나의 토큰으로 인식합니다.<br>
따라서 검색어 "철수"(명사)와 저장된 "철수가"(명사+조사)는 형태소 타입이 달라 검색되지 않습니다.

**즉, 단순히 단어가 일치한다고 검색되는 것이 아니라, 형태소 분석 결과(명사, 동사, 형용사 등)까지 일치해야 검색됩니다.**

## 해결 방법

### 형태소 분석기 적용
한국어 형태소 분석을 통해 단어를 품사별로 분리하여 저장합니다.

**분리 예시:**
```
"철수가 개발을 하고 있습니다."
↓
토큰 분리 결과:
- "철수가" (명사+조사, 원본)
- "철수" (명사)
- "가" (조사)
- "개발을" (명사+조사, 원본)
- "개발" (명사)
- "을" (조사)
- "하고" (동사)
- "있습니다" (동사)
```

이렇게 품사별로 분리된 토큰들을 Elasticsearch에 저장하면 다음과 같은 효과가 있습니다:
- "철수" 검색 → "철수" (명사) 토큰과 매칭되어 검색 성공
- "철수가" 검색 → "철수가" (명사+조사) 토큰과 매칭되어 검색 성공
- "개발" 검색 → "개발" (명사) 토큰과 매칭되어 검색 성공

**핵심은 동일한 품사 형태를 가진 토큰을 만들어 저장하는 것입니다.**

## 형태소 분석기 설정

### 1. Nori 분석기 플러그인 설치
Elasticsearch에서 공식 지원하는 한국어 형태소 분석기입니다.

```bash
# Elasticsearch 설치 경로에서 실행
bin/elasticsearch-plugin install analysis-nori
```

### 2. 인덱스 생성 및 분석기 설정
```json
PUT /search_index
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "nori_user_dict": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed"
        }
      },
      "analyzer": {
        "korean_analyzer": {
          "type": "custom",
          "tokenizer": "nori_user_dict",
          "filter": ["nori_posfilter"]
        }
      },
      "filter": {
        "nori_posfilter": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "J", "SC", "SE", "SF"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "korean_analyzer"
      }
    }
  }
}
```

### 3. 분석기 동작 확인
```json
POST /search_index/_analyze
{
  "analyzer": "korean_analyzer",
  "text": "철수가 개발을 하고 있습니다."
}
```

**결과:**
```json
{
  "tokens": [
    {"token": "철수", "position": 0},
    {"token": "개발", "position": 2},
    {"token": "하", "position": 4}
  ]
}
```

## 검색 쿼리 예시

```json
GET /search_index/_search
{
  "query": {
    "match": {
      "title": "철수"
    }
  }
}
```

이제 "철수"를 검색하면 "철수가 개발을 하고 있습니다."라는 제목이 정상적으로 검색됩니다.

## 주요 설정 옵션

### decompound_mode
- **none**: 복합명사를 분리하지 않음
- **discard**: 복합명사를 분리하고 원본은 버림
- **mixed**: 복합명사를 분리하고 원본도 유지 (권장)

### stoptags (불용어 처리)
- **E**: 동사
- **J**: 조사
- **SC**: 구분자
- **SE**: 줄임표
- **SF**: 마침표, 물음표, 느낌표

## 결론
형태소 분석기를 통해 한국어 검색의 정확도를 크게 향상시킬 수 있습니다.<br>
특히 조사가 붙은 검색어 처리가 필요한 경우 Nori 분석기 적용이 필수적입니다.

