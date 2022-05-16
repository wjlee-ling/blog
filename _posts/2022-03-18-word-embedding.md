---
layout: post
title: 워드 임베딩
subtitle: 언어적 의미를 수치화하기
date: 2022-03-18
last_modified_at: 2022-03-18
tags: [nlp, word embedding, td-idf, word2vec]
---

대학원 의미론 첫 수업 때 교수님은 '의미'가 무엇인지 한시간 얘기하셨다. 사실 언어적 의미가 무엇인가 하는 담론은 지난 한세기가 넘는 시간 동안 지속된 방대한 주제이지만, 컴퓨터로 처리 가능한 의미를 정의하는 것은 또다른 난제다.

*You shall know a word by the company it keeps* (J.R. Firth)
최신의 NLP 모델들은 단어의 의미가 context에서 나온다고 (혹은 크게 영향을 받는다고) 본다. 즉 어떤 단어의 뜻을 알려면 그 주변 단어들의 뜻을 알아야 한다는 것이다. 

## tf-idf
가장 손쉬운 단어 임베딩은 해당 단어의 사전내 인덱스로 표현하거나, term-document matrix에 따라 document별 빈도수로 단어를 표현하는 것이다. 그러나 단어의 빈도로 embedding을 하는 경우(e.g. term-document matrix)엔 자주 등장할 수밖에 없는 determiner, 연결동사 등은 그 값은 클지 모르나 어휘적 의미는 크지 않다. 이에 단순 빈도를 활용한 임베딩을 보완하는 방법을 고안하게 되었다. 일례로 tf-idf는 해당 단어가 등장하는 문서수를 고려하여 다음과 같은 수식으로 벡터값을 구한다.

![tf-idf](https://miro.medium.com/max/816/1*1pTLnoOPJKKcKIcRi3q0WA.jpeg)

즉 모든 document에 등장하는 단어는 idf가 0이 되어 최종 tf-idf값도 0이 된다.

## word2vec
tf-idf와 같은 단어 임베딩은 해당 단어의 언어학적 의미를 포착한다기 보다는 단순 빈도를 나타낸다는 약점이 있다. 이에 word2vec이라는 새로운 임베딩 방식이 소개되었는데, 이 방식은 두가지 방법으로 훈련되었다.
* **skip gram**
한 단어가 주어졌을 때, 정해진 윈도우내 주변 단어들을 정확히 예측하는 확률을 최대화 하는 것이 목표.


