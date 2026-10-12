---
title: "Depth Anything: Unleashing the Power of Large-Scale Unlabeled Data (CVPR 2024)"
date: 2026-10-12 20:30:00 +0900
categories: [논문리뷰]
tags: [monocular depth estimation, self-training, dinov2]
math: true
description: "구하기 어려운 학습 데이터를 어떻게 해결할 것인가? 정답은 깊이 학습에 Pseudo-label image를 활용 하는 것!"
media_subpath: /assets/img/posts/depth-anything/
---

**CVPR 2024.** [Paper](https://arxiv.org/abs/2401.10891) [Project Page](https://depth-anything.github.io) [Github](https://github.com/LiheYoung/Depth-Anything)<br>
Lihe Yang, Bingyi Kang, Zilong Huang, Xiaogang Xu, Jiashi Feng, Hengshuang Zhao<br>
HKU \| TikTok \| CUHK \| ZJU<br>
7 Apr 2024 (arXiv v2)

핵심 한 줄: 구하기 어려운 학습 데이터를 어떻게 해결할 것인가? 정답은 깊이 학습에 Pseudo-label image를 활용 하는 것!

![Depth Anything Figure 1](fig1.webp)
_출처: 원 논문 Figure 1_

라벨 없는 단안 이미지 62M장을 teacher 모델로 Pseudo-label을 생성. 이를 통해 Student 모델에게 대용량의 학습 데이터를 제공. annotation, strong perturbation, DINOv2 feature alignment등의 방법을 적용한 방법론을 제시. zero-shot relative depth SOTA와 downstream task SOTA를 달성한 논문.

---

## Introduction

"foundation model"의 등장으로 큰 변화를 겪고 있다. 이 모델들은 다양한 downstream 시나리오에서 강한 zero-/few-shot 성능을 보이는데, 그 바탕은 데이터 분포를 충분히 덮는 대규모 학습 데이터이다.

로보틱스, 자율주행, VR 등에 폭넓게 쓰이는 monocular depth estimation (MDE)도 단일 이미지에서 depth를 추정하는 foundation model이 필요하지만, 수천만 장 규모의 depth label 데이터셋을 만들기 어려워 덜 탐구되었다. MiDaS가 여러 labeled 데이터셋을 섞어 학습하는 선구적인 연구를 했지만 data coverage의 한계로 일부 시나리오에서 성능이 크게 무너졌다.

본 논문의 목표는 어떤 상황의 어떤 이미지에서도 고품질 depth를 내는 MDE foundation model이며, 이를 dataset scaling-up 관점에서 접근한다. 저자들은 처음으로 대규모 unlabeled monocular 이미지에 주목하며, 그 장점을 세 가지로 든다.

- **Simple and cheap**: 단안 이미지는 어디에나 있어 특수 장비 없이 쉽게 모을 수 있다.
- **Diverse**: 더 넓은 범위의 장면을 덮어 모델의 일반화와 확장성에 중요하다.
- **Easy to annotate**: 사전학습된 MDE 모델의 feedforward 한 번으로 depth label을 붙일 수 있고, LiDAR보다 dense한 depth map을 얻으며 계산량이 큰 stereo matching도 필요 없다.

1. SA-1B, Open Images, BDD100K 등 공개 데이터셋 8개에서 62M장을 모은다(Unlabeled).
2. labeled 이미지 1.5M장으로 초기 MDE 모델을 학습해 annotation 도구로 쓴다.
3. Unlabeled 이미지는 자동 annotation된 뒤 labeled 이미지와 함께 “self-training” 방식으로 학습된다.

\[문제점1\]<br>
하지만 labeled 이미지가 많은 상황에서 대규모 unlabeled 이미지를 제대로 활용하기 어렵다. 실제로 초기 시도에서 labeled 이미지와 pseudo labeled 이미지를 그대로 합쳐 학습해도 labeled 이미지만 쓴 baseline보다 나아지지 않았다. 저자들은 이런 단순한 self-teaching으로는 얻을 수 있는 추가 지식이 적다고 판단했다.

\[해결책1\]<br>
pseudo label을 학습할 때 student에게 더 어려운 최적화 목표를 준다. Student는 강한 perturbation 속에서 학습하며 robust한 표현을 익히게 된다.

\[문제점2\]<br>
auxiliary semantic segmentation task가 MDE에 도움이 된다는 기존 연구를 따라 시도함. 다만, 모델이 이미 충분히 강하면 추가 이득을 얻기 어려웠다. 이미지를 discrete class space로 decoding하면서 semantic 정보가 크게 손실된다고 추측함.

\[해결책2\]<br>
대신 DINOv2의 풍부한 semantic prior를 간단한 feature alignment loss로 유지시킨다. 이는 MDE 성능을 높일 뿐 아니라 middle-level과 high-level perception 모두에 쓰이는 multi-task encoder도 만들어 낸다.

Contribution은 다음과 같다.

- 대규모의 값싸고 다양한 unlabeled 이미지를 MDE에 쓰는 data scaling-up의 가치를 강조한다.
- 대규모 labeled와 unlabeled 이미지를 함께 학습할 때의 핵심 방법을 제시한다. Unlabeled 이미지를 그대로 학습하는 대신 더 어려운 최적화 목표로 추가 지식을 얻게 한다.
- Auxiliary semantic segmentation task 대신 사전학습 encoder의 풍부한 semantic prior를 물려받게 한다.
- metric depth로 fine-tuning하면 ZoeDepth를 크게 앞선다.

---

## Related Work

**Monocular depth estimation.** 딥러닝 기반 방법들은 정교하게 label된 데이터셋에서 depth 표현을 효과적으로 학습했지만, 처음 보는 domain으로 일반화하기 어렵다.

**Zero-shot depth estimation.** MiDaS는 affine-invariant loss로 데이터셋마다 다른 depth scale과 shift를 무시하여 multi-dataset joint training을 가능하게 했고, 그 결과 relative depth를 제공한다. 최근 metric depth를 직접 추정하는 방법들도 나왔지만 저자들의 실험에서는 MiDaS보다 일반화가 떨어졌다. 한편 ZoeDepth가 보였듯 강한 relative depth 모델에 metric depth로 fine-tuning하면 일반화되는 metric depth estimation도 잘 된다. 따라서 relative depth는 MiDaS를 따르되, 대규모 unlabeled 이미지로 강화했다.

**Leveraging unlabeled data.** Semi-supervised learning 분야에 속하지만, 기존 연구는 대부분 labeled 이미지가 적은 상황만 가정한다. Labeled 이미지가 이미 충분하면서 그보다 훨씬 많은 unlabeled 이미지가 있는, 어렵지만 현실적인 상황은 거의 다루지 않았다.

## Depth Anything

![Depth Anything pipeline](fig2.webp)
_출처: 원 논문 Figure 2_

Labeled 집합 $$\mathcal{D}^l = \{(x_i, d_i)\}_{i=1}^M$$과 unlabeled 집합 $$\mathcal{D}^u = \{u_i\}_{i=1}^N$$을 모두 사용한다. 위 그림처럼 먼저 $$\mathcal{D}^l$$로 teacher 모델 $$T$$를 학습하고, $$T$$로 $$\mathcal{D}^u$$에 pseudo depth label을 붙인 뒤, 두 집합을 합쳐 student 모델 $$S$$를 학습한다. 실선은 labeled 이미지, 점선은 unlabeled 이미지의 흐름이고 S 표시는 strong perturbation을 뜻한다.

### 1. Learning Labeled Images

Depth 값 $$t$$를 disparity $$d = 1/t$$로 바꾼 뒤 각 depth map에서 0~1로 정규화한다. 데이터셋마다 알 수 없는 scale과 shift를 무시하기 위해 affine-invariant loss를 쓴다.

$$
\mathcal{L}_l = \frac{1}{HW}\sum_{i=1}^{HW}\rho(d_i^*, d_i), \qquad \rho(d_i^*, d_i) = \lvert\hat{d}_i^* - \hat{d}_i\rvert
$$

여기서 $$d_i^*$$와 $$d_i$$는 각각 예측과 ground truth이고, $$\hat{d}$$는 translation 0, scale 1이 되도록 맞춘 값이다.

$$
\hat{d}_i = \frac{d_i - t(d)}{s(d)}, \qquad t(d) = \mathrm{median}(d), \quad s(d) = \frac{1}{HW}\sum_{i=1}^{HW}\lvert d_i - t(d)\rvert
$$

![Labeled and unlabeled datasets](table1.webp){: w="480" }
_출처: 원 논문 Table 1_

사용한 Labeled / Unlabeled 데이터 셋이다.

### 2. Unleashing the Power of Unlabeled Images

1. Teacher $$T$$로 unlabeled 집합의 pseudo label 집합을 만든다.

   $$
   \hat{\mathcal{D}}^u = \{(u_i, T(u_i)) \mid u_i \in \mathcal{D}^u\}_{i=1}^N
   $$

2. Labeled와 pseudo labeled를 합친 $$\mathcal{D}^l \cup \hat{\mathcal{D}}^u$$로 student $$S$$를 학습하며, $$T$$에서 fine-tuning하지 않고 $$S$$를 다시 초기화하는 것이 성능이 더 좋다.

**Self-training만으로는 개선되지 않았다.** 저자들의 추측은 두 가지이다.

- Labeled 이미지가 이미 충분해서 unlabeled 이미지에서 얻는 추가 지식이 제한적.
- Teacher와 student가 같은 사전학습과 architecture를 공유해 unlabeled 집합에서 비슷한 정답과 오답을 내므로, student가 새로 배우는 것이 없다.

**해결: student에게 strong perturbation을 준다.** unlabeled 이미지에 강한 perturbation을 넣어, student가 추가 시각 지식을 적극적으로 찾고 invariant한 표현을 배우게 한다.

- **Strong color distortion**: color jittering, Gaussian blurring
- **Strong spatial distortion**: CutMix

**(CutMix)** 무작위 unlabeled 이미지 쌍 $$u_a$$, $$u_b$$를 공간적으로 섞는다. $$M$$은 사각형 영역이 1인 binary mask이다.

$$
u_{ab} = u_a \odot M + u_b \odot (1 - M)
$$

Unlabeled loss $$\mathcal{L}_u$$는 $$M$$과 $$1-M$$이 정하는 영역에서 각각 affine-invariant loss를 계산한 뒤, 영역 넓이로 가중 평균한다.

$$
\mathcal{L}_u^M = \rho\big(S(u_{ab}) \odot M,\ T(u_a) \odot M\big), \qquad \mathcal{L}_u^{1-M} = \rho\big(S(u_{ab}) \odot (1-M),\ T(u_b) \odot (1-M)\big)
$$

$$
\mathcal{L}_u = \frac{\sum M}{HW}\mathcal{L}_u^M + \frac{\sum (1-M)}{HW}\mathcal{L}_u^{1-M}
$$

쉽게 말해, 학생 모델한테 두 이미지를 합쳐서 제공. 첫번째 영역 M에 대해서 선생의 예측과 비교. 두 번째 영역 1-M에 대해서 선생의 예측과 비교. 이 두 영역에 대한 Loss의 합이라고 보면 된다.

### 3. Semantic-Assisted Perception

High-level semantic 정보는 depth 모델에 도움이 된다. unlabeled 이미지를 쓰는 상황에서는 다른 task의 auxiliary supervision이 pseudo depth label의 noise를 상쇄하는 역할도 할 수 있다.

하늘이 어디 부분인지 알면 해당 부분은 거리가 멀다고 당연히 어느 정도 알 수 있음!

**① Semantic segmentation label 부여 (실패).** RAM + GroundingDINO + HQ-SAM을 조합해 unlabeled 이미지에 semantic segmentation label을 붙였다. 다만, 이미지를 discrete class space로 decoding 하면서 semantic 정보가 너무 많이 손실됨.(4K 클래스)

**② DINOv2 feature alignment.** 더 풍부한 semantic 신호를 찾던 중, DINOv2가 fine-tuning 없이 frozen 가중치만으로도 강한 semantic task 성능을 보이는 점에 주목하였다. 하여 auxiliary feature alignment loss로 DINOv2의 semantic 능력을 depth 모델에 부여한다.

$$
\mathcal{L}_{feat} = 1 - \frac{1}{HW}\sum_{i=1}^{HW}\cos(f_i, f'_i)
$$

$$f$$는 depth 모델 $$S$$의 feature, $$f'$$는 frozen DINOv2 encoder의 feature이다. 코사인 유사도 = 두 feature가 동일하면 Cos값은 1이다.

**Tolerance margin $$\alpha$$.** Segmentation의 능력이 깊이 추정에 도움이 되는 것만은 아님. 자동차에 앞면과 뒷 부분은 깊이가 다르다. 하지만 Segmentation은 동일하게 여기므로 두 feature를 맞춰주는데 마진을 둔다. 적당히 비슷하면 로스에 반영 X.

최종 loss는 $$\mathcal{L}_l$$, $$\mathcal{L}_u$$, $$\mathcal{L}_{feat}$$ 세 loss의 평균이다.

---

## Results

- 자세한 실험 설정은 논문 참고

### 1. Zero-Shot Relative Depth Estimation

![Zero-shot relative depth estimation](table2.webp)
_출처: 원 논문 Table 2_

처음 보는 6개 데이터셋 (KITTI, NYUv2, Sintel, DDAD, ETH3D, DIODE)에서 zero-shot 성능을 평가. 지표는 AbsRel ($$\lvert d^* - d\rvert/d$$)과 δ1 ($$\max(d^*/d, d/d^*) < 1.25$$인 픽셀 비율)이다.

- 둘 다 ViT-L일 때 모든 데이터셋에서 MiDaS를 앞선다.
- ViT-B 모델만으로도 훨씬 큰 ViT-L 기반 MiDaS보다 확실히 낫다.
- MiDaS v3.1은 KITTI와 NYUv2의 학습 이미지를 사용해 엄밀한 zero-shot이 아닌데도, 이들 이미지를 전혀 쓰지 않은 Depth Anything이 더 좋다.

### 2. Fine-tuned to Metric Depth Estimation

metric depth estimation의 성능도 평가. Encoder를 Depth Anything ViT-L로 초기화하고 decoder는 무작위 초기화한 뒤 metric depth로 fine-tuning한다.

- **In-domain**: 같은 domain에서 학습하고 평가
- **Zero-shot**: 한 domain (예: NYUv2)에서 학습하고 다른 domain (예: SUN RGB-D)에서 평가

![Fine-tuning on NYUv2](table3.webp){: w="480" }
_출처: 원 논문 Table 3_

![Fine-tuning on KITTI](table4.webp){: w="480" }
_출처: 원 논문 Table 4_

![Zero-shot metric depth estimation](table5.webp)
_출처: 원 논문 Table 5_

Zero-shot: 실내외 모든 데이터셋에서 원래 ZoeDepth보다 좋다.

### 3. Fine-tuned to Semantic Segmentation

MDE encoder를 semantic segmentation 데이터셋에 fine-tuning해 semantic 능력을 확인한다.

![Transfer to Cityscapes](table7.webp){: w="480" }
_출처: 원 논문 Table 7_

![Transfer to ADE20K](table8.webp){: w="480" }
_출처: 원 논문 Table 8_

Cityscapes에서 대규모 MDE 학습으로 얻은 encoder가 86.2 mIoU (multi-scale)로, ImageNet-21K로 대규모 사전학습한 Swin-L (84.3), ConvNeXt-XL (84.6)보다 좋다. ADE20K에서도 Mask2Former를 segmentation 모델로 써서 이전 최고 58.3을 59.4로 올린다. 저자들은 이 encoder가 middle-level과 high-level perception 모두에 쓰이는 범용 multi-task encoder가 될 수 있다고 본다.

### 4. Ablation Studies

![Ablation of strong perturbation and semantic constraint](table9.webp){: w="480" }
_출처: 원 논문 Table 9_

**Strong perturbation과 semantic constraint의 효과.** Pseudo label이 붙은 unlabeled 이미지를 단순히 추가하는 것 ($$\mathcal{L}_u$$)은 성능을 거의 올리지 못한다. 그러나 strong perturbation ($$\mathcal{S}$$)을 넣으면 student가 추가 시각 지식을 찾게 되어 모든 데이터셋에서 좋아지고, semantic constraint ($$\mathcal{L}_{feat}$$)를 더하면 unlabeled 이미지의 효과가 더 커진다.

![Comparison with MiDaS encoder](table10.webp){: w="480" }
_출처: 원 논문 Table 10_

**MiDaS encoder와의 비교.** Downstream fine-tuning에서도 Depth Anything encoder가 MiDaS v3.1 encoder보다 depth estimation과 semantic segmentation 모두에서 확실히 좋다.

![Comparison with DINOv2 encoder](table11.webp){: w="480" }
_출처: 원 논문 Table 11_

**DINOv2와의 비교.** Downstream metric depth와 semantic segmentation 모두에서 원래 DINOv2보다 좋아, DINOv2가 이미 강한 초기값이지만 대규모 고품질 MDE 학습이 downstream 전이 성능을 더 끌어올린다.

---

## Conclusion

본 논문은 robust monocular depth estimation을 위한 실용적인 해법 Depth Anything을 제안함. 값싸고 다양한 unlabeled 이미지의 가치를 강조 + 그 가치를 끌어내기 위한 간단하지만 효과적인 두 전략을 설계.

(1) Unlabeled 이미지를 학습할 때 strong perturbation을 주자.

(2) 사전학습 모델의 풍부한 semantic prior를 활용하자.
