---
title: "Vision Transformers Need Registers (ICLR 2024)"
date: 2026-11-16 20:30:00 +0900
categories: [논문리뷰]
tags: [vision transformer, self-supervised learning, dinov2, attention]
description: "큰 모델에서 layer를 거듭하면서 local feature가 손실되고 global feature로 채워진다. 이에 따라 outlier가 등장. 이를 막기 위해 Register token을 도입하여 정보를 보관하는 방법론을 제시."
media_subpath: /assets/img/posts/vit-registers/
---

**ICLR 2024.** [Paper](https://arxiv.org/abs/2309.16588) [Github](https://github.com/facebookresearch/dinov2)<br>
Timothée Darcet, Maxime Oquab, Julien Mairal, Piotr Bojanowski<br>
Meta FAIR \| Univ. Grenoble Alpes, Inria<br>
12 Apr 2024 (arXiv v2)

핵심 한 줄: 큰 모델에서 layer를 거듭하면서 local feature가 손실되고 global feature로 채워진다. 이에 따라 outlier가 등장. 이를 막기 위해 Register token을 도입하여 정보를 보관하는 방법론을 제시.

![Attention maps with and without registers](fig1.webp)
_출처: 원 논문 Figure 1_

최근 ViT의 feature map과 attention map에는 배경에 튀는 artifact가 있다. 저자들은 큰 모델이 충분히 학습되면 이웃과 비슷한 patch를 골라 global 정보를 저장하는 데 재활용한 결과라고 분석한다. 해결책은 간단하다. 입력 sequence에 학습 가능한 register token을 몇 개 추가하고 출력에서 버린다. 이것만으로 세 학습 방식 모두에서 artifact가 사라지고, dense prediction 성능이 오른다.

---

## Introduction

특정 task용 annotation을 모으기 어렵기 때문에, 데이터가 많은 task로 사전학습한 모델의 일부를 feature extractor로 쓰는 방식이 흔하다. Classification 같은 supervised 학습, CLIP 같은 text-image alignment, 그리고 Transformer 기반 self-supervised 학습이 대표적이다.

그중 DINO는 마지막 attention layer가 이미지에서 의미 있는 부분에 자연스럽게 집중해, 해석하기 쉬운 attention map을 낸다. LOST 같은 object discovery 방법은 이 성질을 이용해 라벨 없이 물체를 찾는다.

DINOv2는 frozen backbone + linear 모델만으로 monocular depth estimation과 semantic segmentation을 잘 해낼 만큼 dense feature가 좋다. 그런데 이상하게도 LOST와는 잘 맞지 않아, supervised backbone 수준의 실망스러운 성능을 낸다. 조사해 보니 DINOv2의 feature map에는 DINO에 없던 artifact가 있었고, DeiT-III와 OpenCLIP에도 같은 artifact가 있었다. 오히려 DINO가 예외였던 것이다. (Fig1에서 보이듯 노란색 이상치가 artifact이다.)

1. Artifact는 출력 norm이 약 10배 큰 token이며, 전체 sequence의 약 2%를 차지한다.
2. ViT의 중간 layer쯤에서 생기고, 충분히 큰 모델을 충분히 오래 학습해야 나타난다.
3. 이웃 patch와 비슷한, 즉 추가 정보가 거의 없는 patch에 생긴다.
4. 이 token은 원래 위치와 픽셀 정보는 적게, 이미지 전체에 대한 global 정보는 많이 담고 있다.
5. 해석: 모델이 쓸모가 적은 patch를 알아보고, 그 token을 global 정보 집계용으로 재활용하면서 공간 정보는 버린다.
6. 이를 확인하기 위해 입력 이미지와 무관한 token(register)을 sequence에 추가하면, outlier token이 완전히 사라지고 dense prediction 성능과 object discovery가 좋아진다.

\[문제점\]<br>
큰 ViT의 feature map과 attention map에 배경 쪽으로 튀는 high-norm artifact가 있다. Attention map을 해석하기 어렵고, feature map이 매끄러워야 동작하는 object discovery(LOST)가 망가진다.

\[원인\]<br>
모델이 정보가 중복된 patch token을 내부 계산용으로 재활용한다. 그 결과 해당 patch의 local 정보가 버려진다.

\[해결책\]<br>
Global 계산 자체가 나쁜 것은 아니고, 그것이 patch token 안에서 일어나는 것이 문제다. 그래서 입력에 register token을 추가해 이 역할을 따로 맡기고, 출력에서는 버린다.

Contribution은 다음과 같다.

- Label·text·self-supervised로 학습한 ViT 모두에서 artifact를 찾아내고, 이를 출력 norm이 튀는 outlier token으로 정량화한다.
- Outlier token이 언제, 어디에 생기고, 어떤 정보를 담는지 분석해 '중복된 patch를 global 계산용으로 재활용한다'는 해석을 제시한다.
- 학습 가능한 register token을 추가하는 간단한 해결책으로 artifact를 완전히 없애고, self-supervised 모델의 dense prediction SOTA, 큰 모델로도 가능한 object discovery, 더 매끄러운 feature·attention map을 얻는다.

---

## Problem Formulation

![Attention maps of supervised, text-supervised and self-supervised ViTs](fig2.webp)
_출처: 원 논문 Figure 2_

Label supervision(DeiT-III), text supervision(OpenCLIP), self-supervision(DINO, DINOv2)으로 학습한 ViT의 attention map. DINO를 뺀 모든 모델에서 배경에 튀는 값이 보인다. 이 절에서는 artifact가 왜, 언제 생기는지를 DINOv2 중심으로 분석한다.

### 1. Artifact = high-norm outlier token

![Norm distribution of local features](fig3.webp)
_출처: 원 논문 Figure 3_

Artifact patch와 다른 patch의 가장 큰 차이는 모델 출력에서 token embedding의 norm이다. DINOv2 ViT-g의 local feature norm 분포는 뚜렷하게 둘로 갈라진다(bimodal). 그래서 norm이 150보다 큰 token을 "high-norm" token으로 정의하고(기준값은 모델마다 다름), 이후 "outlier"와 같은 뜻으로 쓴다. DINOv2에서 이런 token은 2.37%이며, DINO ViT-B/16에는 없다.

### 2. 언제 생기나: 큰 모델, 충분한 학습

![When high-norm tokens appear](fig4.webp)
_출처: 원 논문 Figure 4_

- (a) 40-layer ViT-g에서 약 15번째 layer부터 다른 patch와 갈라진다.
- (b) 전체 학습의 약 1/3이 지난 뒤부터 나타난다.
- (c) Tiny부터 giant까지 6개 크기 중 큰 세 모델에서만 나타난다.

참고로 MAE에는 artifact가 없다. Patch 단위 local loss만 써서 global 정보를 모을 필요가 없기 때문으로 저자들은 추정한다 (Appendix E).

### 3. 어디에 생기고, 무엇을 담나

![Similarity and local information of outlier tokens](fig5.webp)
_출처: 원 논문 Figure 5_

**중복된 patch에 생긴다.** Patch embedding 직후 각 patch와 이웃 4개 사이의 cosine similarity를 보면, high-norm token은 이웃과 매우 비슷한 patch에서 생긴다 (Fig. 5a). 정보가 중복되어 버려도 표현 품질에 손해가 적은 곳, 즉 균일한 배경이다. 또한 이미지 중심보다 가장자리에 더 자주 생기는데, 사람이 찍은 사진은 물체가 가운데 있어 가장자리가 대개 배경이기 때문으로 본다 (Appendix A).

**Local 정보가 적다.** Patch embedding 위에 linear model을 학습해 비교했다 (Fig. 5b).

- Position prediction: 위치 예측 top-1이 normal 41.7, outlier 22.8. 평균 거리 오차는 0.79 vs 5.09. 이 위치 정보는 첫 layer 전에 absolute position embedding으로 넣어 준 것이다.
- Pixel reconstruction: 원래 픽셀 복원 L2 error가 18.38 vs 25.23.

![Linear probing on single tokens](table1.webp)
_출처: 원 논문 Table 1_

**Global 정보를 담는다.** 이미지마다 patch token 하나를 무작위로 골라 그 token 하나로 logistic regression 분류기를 학습했다. Outlier token은 normal token보다 훨씬 정확하고 `[CLS]` token에 가깝다 (예: Aircraft 17.1 → 79.1, `[CLS]` 87.3).

**outlier token은 자기 자리의 그림 정보는 거의 잃은 대신 이미지 전체에 대한 요약을 들고 있다.**

---

## Registers

![Register tokens added to the ViT input](fig6.webp)
_출처: 원 논문 Figure 6_

**가설.** 크고 충분히 학습된 모델은 중복된 token을 알아보고, 그것을 global 정보를 저장·처리·꺼내 쓰는 자리로 쓴다. 이 행동 자체는 나쁘지 않지만 patch token 안에서 일어나는 것이 문제다. 그 patch의 local 정보를 버리게 되어 dense prediction 성능이 떨어질 수 있다.

**해결.** Patch embedding layer 뒤에 `[CLS]`처럼 학습 가능한 값을 가진 token N개를 sequence에 추가한다. 이것이 register다. Transformer 끝에서 register는 버리고, `[CLS]`와 patch token만 image representation으로 쓴다 (학습·추론 모두).

**모델이 원래 몰래 배경 patch에서 하던 계산을 위해 전용 메모장을 따로 주는 것이다.**

- 어떤 학습 요소가 artifact를 만드는지는 완전히 밝히지 못했다. Pretraining 방식(OpenCLIP과 DeiT-III은 B, L 크기 모두 outlier가 있음), 모델 크기, 학습 길이가 모두 관여한다.
- 비용: register 4개면 FLOPs 증가가 2% 미만, parameter 증가는 무시할 수준이다 (16개면 FLOPs 최대 6%, Appendix B).

---

## Results

- 자세한 실험 설정은 논문 참고
- 세 학습 방식에 구조 변경만 그대로 적용: DeiT-III(label supervision, ImageNet-22k, ViT-B), OpenCLIP(text supervision, Shutterstock 기반 라이선스 이미지-텍스트, ViT-B/16), DINOv2(self-supervised, ImageNet-22k, ViT-L).
- 따로 적지 않으면 register는 4개.

### 1. Artifact 제거

![Norm distribution with and without registers](fig7.webp)
_출처: 원 논문 Figure 7_

모델 출력 feature의 norm 분포. Register를 넣고 학습하면 세 모델 모두 large-norm token이 사라진다.

### 2. Downstream 성능

![Downstream evaluation with and without registers](table2.webp)
_출처: 원 논문 Table 2_

- Register를 넣어도 성능이 떨어지지 않고, 오히려 좋아지는 경우가 있다.
- DINOv2: ImageNet 84.3 → 84.8, ADE20k 46.6 → 47.9, NYUd rmse 0.378 → 0.366.
- OpenCLIP: NYUd rmse 0.702 → 0.661, zero-shot ImageNet 59.9 → 60.1 (학습 데이터 출처가 달라 절대 성능은 원래 OpenCLIP보다 낮다).
- DeiT-III: 거의 변화 없음.

### 3. Register 개수

![Ablation on the number of registers](fig8.webp)
_출처: 원 논문 Figure 8_

DINOv2 ViT-L/14를 register 0, 1, 2, 4, 8, 16개로 학습했다.

- Register를 1개만 넣어도 눈에 보이는 artifact가 사라진다.
- Dense task(segmentation, depth)에는 최적 개수가 있는 것으로 보이며, 1개가 이득의 대부분을 준다. Artifact가 사라져 local feature가 좋아진 효과로 해석한다.
- ImageNet은 register가 많을수록 계속 좋아진다.
- 모든 실험은 4개로 고정했다.

### 4. Object Discovery

![Unsupervised object discovery with LOST](table3.webp)
_출처: 원 논문 Table 3_

LOST로 PASCAL VOC 2007·2012, COCO 20k에서 unsupervised object discovery를 평가했다 (corloc). DeiT-III와 OpenCLIP은 value, DINOv2는 key feature를 쓴다.

- DINOv2는 35.3 → 55.4 (VOC 2007), DeiT-III는 11.7 → 27.1로 크게 좋아진다.
- OpenCLIP은 오히려 조금 떨어진다 (38.8 → 37.1). OpenCLIP은 value projection이 outlier를 이미 걸러내서, register가 없어도 LOST score가 깨끗하기 때문이다 (Appendix C).
- 그래도 DINOv2+reg(55.4)는 DINO(61.9)에는 못 미친다.

### 5. Register의 attention

![Attention maps of the CLS token and registers](fig9.webp)
_출처: 원 논문 Figure 9_

`[CLS]`와 register에서 patch로 가는 attention map. Register들의 행동이 완전히 같지 않고, 일부는 장면 속 서로 다른 물체에 주목한다. Slot attention과 비슷한 모습인데, 강제한 적 없는 행동이 학습 중에 자연스럽게 생겼다.

### 6. Register가 outlier의 역할을 넘겨받는다 (Appendix D)

![Norm distribution of patch tokens and registers](fig15.webp)
_출처: 원 논문 Figure 15_

Register가 있으면 patch token의 norm에는 outlier가 없고, high-norm은 모두 register 쪽에 있다. Outlier를 만들던 행동을 register가 흡수한 것이다.

![Linear probing of CLS, patch and register tokens](table4.webp)
_출처: 원 논문 Table 4_

Aircraft linear probing. Register를 넣어도 `[CLS]`와 normal patch의 점수는 비슷하고, outlier patch(73.3)가 하던 global 정보 집계를 register(71.1)가 대신한다. Normal patch의 local 정보도 그대로다.

---

## Conclusion

DINOv2를 비롯한 여러 인기 ViT의 feature map에서 artifact를 찾아냈다. 또한 위치를 분석해, 모델이 정보가 적은 영역의 token을 다른 역할로 재활용한다고 해석했다. 이에 따라 출력으로 쓰지 않는 token을 입력 sequence에 덧붙이는 간단한 해결책을 제안했고, artifact가 완전히 사라지면서 dense prediction과 object discovery 성능이 좋아졌다.
