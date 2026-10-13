---
title: "Depth Anything V2 (NeurIPS 2024)"
date: 2026-10-13 15:40:00 +0900
categories: [논문리뷰]
tags: [monocular depth estimation, synthetic data, pseudo label]
math: true
description: "합성 데이터의 장점을 활용하여 Fine detail과 robustness를 얻음. 정밀한 synthetic 이미지로 가장 큰 teacher를 학습하고, 그 teacher가 pseudo label을 붙인 대규모 real 이미지로 student를 학습."
media_subpath: /assets/img/posts/depth-anything-v2/
---

**NeurIPS 2024.** [Paper](https://arxiv.org/abs/2406.09414) [Project Page](https://depth-anything-v2.github.io) [Github](https://github.com/DepthAnything/Depth-Anything-V2)<br>
Lihe Yang, Bingyi Kang, Zilong Huang, Zhen Zhao, Xiaogang Xu, Jiashi Feng, Hengshuang Zhao<br>
HKU \| TikTok<br>
20 Oct 2024 (arXiv v2)

핵심 한 줄: 합성 데이터의 장점을 활용하여 Fine detail과 robustness를 얻음. 정밀한 **synthetic** 이미지로 가장 큰 teacher를 학습하고, 그 teacher가 pseudo label을 붙인 대규모 **real** 이미지로 student를 학습.

![Depth Anything V2 Figure 1](fig1.webp)
_출처: 원 논문 Figure 1_

(1) labeled real 이미지를 전부 synthetic 이미지로 교체, (2) teacher를 DINOv2-G로 키움, (3) 62M장의 pseudo-labeled real 이미지를 이용해 student를 학습. 가장 작은 모델(25M)이 60ms로 SD 기반 Marigold(948M, 5.2s)보다 정확하고, 25M~1.3B 네 가지 크기를 제공한다. 기존 test set의 noise와 낮은 diversity를 보완하는 benchmark DA-2K도 제안.

---

## Introduction

Monocular depth estimation (MDE)은 3D reconstruction, navigation, 자율주행 같은 고전적인 응용뿐 아니라 이미지·영상·3D scene 생성 같은 AI-generated content에서도 쓰인다. 그래서 open-world 이미지를 다루는 MDE 모델이 많이 나왔고, 구조로 보면 두 갈래로 나뉜다.

- **Discriminative models** (MiDaS, Depth Anything V1 등. BEiT, DINOv2 기반): complex scene에서 robust하고 빠르고 가볍지만, fine detail이 부족하고 transparent object와 reflection에 약하다.
- **Generative models** (Marigold 등. Stable Diffusion 기반): 세밀한 detail 표현은 뛰어나지만 느리고 무겁다.

![Robustness of V1 and fine detail of Marigold](fig2.webp)
_출처: 원 논문 Figure 2_

첫 줄은 V1의 robustness (헷갈리는 방 구조), 둘째 줄은 Marigold의 fine detail (얇은 농구 그물).

![Preferable properties of an MDE model](table1.webp)
_출처: 원 논문 Table 1_

본 논문의 목표는 Table 1의 장점을 모두 갖춘 MDE foundation model이다.

- complex layout, transparent object (유리), reflective surface (거울, 화면)에서도 robust한 예측
- 얇은 물체 (의자 다리), 작은 구멍까지 담는 Marigold 수준의 fine detail
- 다양한 model scale과 빠른 inference
- downstream task로 fine-tuning할 수 있는 transferability

이처럼 V1의 장점은 유지하고 단점을 보완하자는 것이 저자들의 목표이다. 저자들은 이를 위해 화려한 기법은 필요 없고 **핵심은 여전히 데이터**라고 말한다. V1이 unlabeled 데이터로 data coverage를 넓혔다면, 이번에는 labeled 데이터 설계부터 다시 본다.

1. **Labeled real 이미지**를 모두 정밀한 **synthetic 이미지** (595K장)로 바꾼다.
2. **Synthetic 이미지만으로** 가장 큰 DINOv2-G 기반 teacher를 학습한다.
3. Teacher로 **unlabeled “real” 이미지** 62M장에 pseudo depth label을 붙인다.
4. Pseudo-labeled real 이미지만으로 student (ViT-S/B/L/G)를 학습한다.

\[문제점1\]<br>
Real labeled data는 depth sensor, stereo matching, SfM의 한계로 **label noise가 있고, 경계나 얇은 구조 같은 detail을 놓친다**. 이걸로 학습한 모델도 같은 실수를 한다.

\[해결책1\]<br>
Labeled real 이미지를 **전부 정밀한 synthetic 이미지로 교체**한다. 무거운 diffusion 없이 discriminative 모델로도 아주 세밀한 detail을 얻을 수 있다.

\[문제점2\]<br>
Synthetic 이미지는 real 이미지와 **distribution shift가 있고 scene coverage가 좁아** real 이미지로 일반화하기 어렵다. 가장 큰 DINOv2-G만 그럭저럭 일반화하지만 여전히 하늘이나 사람에서 실패하고, 1.3B라 배포에 어려움이 있다.

\[해결책2\]<br>
Synthetic으로 학습한 가장 큰 teacher가 대규모 unlabeled real 이미지에 pseudo label을 붙이고, student들은 이 pseudo-labeled **real** 이미지로 학습한다. **Real** 이미지가 domain gap을 메우는 다리가 되고, teacher의 능력이 label 수준에서 작은 모델로 옮겨진다.

\[문제점3\]<br>
기존 test set (NYU-D 등)도 label noise가 많고 scene이 단조로우며 해상도가 낮아, **V2가 개선한 fine detail과 robustness가 수치에 드러나지 않는다**.

\[해결책3\]<br>
정밀한 sparse annotation과 다양한 scene을 갖춘 새 평가 benchmark **DA-2K**를 만든다.

Contribution은 다음과 같다.

- Labeled real 이미지를 정밀한 synthetic 이미지로 전부 바꾸는 것이 fine detail의 핵심임을 보인다.
- Synthetic 이미지의 단점을 가장 큰 teacher와 대규모 pseudo-labeled real 이미지로 해결하는 학습 방법을 제시한다.
- SD 기반 모델보다 10배 이상 빠르고 더 정확한 25M~1.3B 크기의 모델을 제공하고, metric depth로 fine-tuning한 모델도 제공한다.
- 다양한 scene과 정밀한 annotation을 갖춘 평가 benchmark DA-2K를 제안한다.

---

## Related Work

**Monocular depth estimation.** 초기 연구는 in-domain metric depth에 집중했지만, 쓸 수 있는 곳이 좁아 최근에는 zero-shot relative depth가 주목 받는다. 일부는 Stable Diffusion을 depth denoiser로 쓰는 등 modeling 방식을 개선했고 (Marigold, DepthFM, GeoWizard), 일부는 데이터 관점에서 접근했다 (MiDaS 2M장, Metric3D 8M장의 labeled 이미지, V1의 62M장 unlabeled 이미지). 본 논문은 labeled real 이미지의 한계를 지적하고 synthetic 이미지로 정밀도를 확보하며, 그로 인한 일반화 문제는 데이터 (pseudo-labeled real 이미지)와 모델 (teacher 크기 키우기) 양쪽으로 푼다.

**Learning from unlabeled real images.** Semi-supervised learning 연구는 대부분 작은 규모의 academic benchmark만 다룬다. 본 논문은 0.6M장 labeled 이미지 baseline을 62M장 unlabeled 이미지로 끌어올리는 현실적인 상황을 다루고, 특히 labeled real 이미지를 synthetic으로 바꿨을 때 unlabeled real 이미지가 필수임을 보인다.

**Knowledge distillation.** 가장 큰 teacher의 지식을 작은 모델로 옮긴다는 점은 KD와 같다. 다만 KD는 labeled 이미지에서 feature나 logit 수준으로 distillation하는 반면, 본 논문은 추가 unlabeled real 이미지를 통해 prediction (label) 수준에서 한다. Scale 차이가 큰 두 모델 사이의 feature distillation은 위험한데, pseudo label distillation은 1.3B에서 25M으로도 쉽고 안전하다.

---

## Depth Anything V2

### 1. Revisiting Labeled Data: Real → Synthetic

저자들은 labeled real 이미지가 정말 도움이 되는지 되묻고, 두 가지 단점을 든다.

- **Label noise**: depth sensor는 transparent object의 depth를 못 잡고, stereo matching은 textureless나 반복 패턴에 약하고, SfM은 dynamic object와 outlier에 약하다.
- **Ignored details**: 경계나 얇은 구멍의 depth가 거칠게 기록되어 있어, 이걸로 학습하면 over-smoothed 예측이 나온다.

![Label noise in real depth datasets](fig3.webp)
_출처: 원 논문 Figure 3_

(a)~(c)는 각 수집 방식의 label noise, (d)는 그렇게 학습한 모델이 같은 실수를 하는 모습. 실제로 MiDaS와 V1은 Transparent Surface Challenge에서 25.9%, 53.5%에 그친다 (V2는 zero-shot으로 83.6%).

![Depth labels of real and synthetic images](fig4.webp)
_출처: 원 논문 Figure 4_

Synthetic 이미지의 depth label은 두 가지 면에서 정밀하다.

1. 경계, 얇은 구멍, 작은 물체 같은 fine detail이 모두 정확하다. (b)의 얇은 그물 구조와 나뭇잎까지 진짜 depth가 붙어 있다.
2. 어려운 transparent object와 reflective surface의 실제 depth도 얻을 수 있다 ((b)의 테이블 위 꽃병).

즉 **synthetic 이미지의 depth는 진짜 "GT"**이고, (c) 오른쪽처럼 이걸로 학습한 모델도 세밀하게 예측한다. 그래픽 엔진에서 빠르게 늘릴 수 있고 real 이미지와 달리 개인정보나 윤리 문제도 없다.

### 2. Challenges in Using Synthetic Data

그렇다면 왜 여전히 real 데이터가 MDE의 주류인가? Synthetic 이미지에는 두 가지 한계가 있다.

- **Limitation 1. Distribution shift**: 그래픽 엔진이 생성한 synthetic 이미지는 색이 너무 "clean"하고 layout이 너무 "ordered"하다. Real 이미지는 훨씬 무작위적이라 synthetic에서 real로 바로 옮기기 어렵다.
- **Limitation 2. Restricted scene coverage**: "living room", "street scene"처럼 미리 정해진 scene에서 반복 샘플링한다. Hypersim, Virtual KITTI가 아무리 정밀해도 "crowded people" 같은 real 장면으로 일반화를 기대하기 어렵다.

![Synthetic-to-real transfer of vision encoders](fig5.webp)
_출처: 원 논문 Figure 5_

BEiT, SAM, SynCLR, DINOv2 네 가지 사전학습 encoder로 synthetic 이미지만 학습해 본 pilot study. **DINOv2-G만** 만족스러운 결과를 낸다.

그럼 가장 큰 DINOv2-G를 쓰면 될까? 또 두 가지 문제가 있다.

![Failure cases of DINOv2-G trained on synthetic images](fig6.webp)
_출처: 원 논문 Figure 6_

- Synthetic 학습 데이터에 드문 패턴은 DINOv2-G도 틀린다. 하늘 (구름)은 아주 멀어야 하는데 가깝게, 사람 머리는 몸과 depth가 맞지 않게 예측한다. Synthetic 학습 셋에 다양한 하늘이나 사람이 없기 때문이다.
- 1.3B인 DINOv2-G는 저장 공간과 inference 속도 면에서 대부분의 응용에 너무 무겁다.

다른 대안도 마땅치 않다. Real과 synthetic을 섞어 학습하면 synthetic의 장점이 사라진다. 추가로 synthetic 이미지를 모든 상황에 대해 만드는 것은 불가능하다.

### 3. Key Role of Large-Scale Unlabeled Real Images

해결책은 단순하다. **Unlabeled real 이미지**를 넣는다. Synthetic 이미지로만 학습한 DINOv2-G teacher가 unlabeled real 이미지에 pseudo depth label을 붙이고, 새 모델은 이 pseudo-labeled 이미지로만 학습한다. 저자들은 unlabeled real 이미지의 역할을 세 가지로 정리한다.

- **Bridge the domain gap**: Synthetic에서 real 테스트 이미지로 바로 옮기는 대신 real 이미지를 중간 학습 대상으로 두면, 모델이 real-world 분포에 익숙해진다.
- **Enhance the scene coverage**: 공개 데이터셋의 대규모 unlabeled 이미지로 다양한 scene을 쉽게 덮는다. synthetic 이미지와 달리, real 이미지는 서로 다른 정보가 많다.
- **Transfer knowledge from the most capable model to smaller ones**: 작은 모델은 스스로 synthetic-to-real transfer를 못 하지만, pseudo-labeled real 이미지로 가장 큰 모델의 고품질 예측을 따라 할 수 있다. Feature나 logit이 아니라 label 수준의 distillation이라 teacher와 student의 크기 차이가 커도 안전하다.

![Depth Anything V2 pipeline](fig7.webp)
_출처: 원 논문 Figure 7_

### 4. Overall Framework

최종 학습 과정은 세 단계다.

1. DINOv2-G 기반의 teacher를 정밀한 synthetic 이미지로만 학습한다.
2. Teacher로 대규모 unlabeled real 이미지에 정밀한 pseudo depth를 만든다.
3. Student를 pseudo-labeled real 이미지로만 학습한다 (이 단계에서 synthetic 이미지는 필요 없다. Ablation 참고).

Student는 DINOv2 small, base, large, giant 기반 네 가지이고, depth decoder는 V1처럼 DPT를 쓴다.

![Training data sources](table7.webp){: w="480" }
_출처: 원 논문 Table 7_

Synthetic 데이터셋 5개 (595K장)와 pseudo-labeled real 데이터셋 8개 (62M장)를 쓴다.

V1과 같이 affine-invariant inverse depth를 예측하고, pseudo label에서 loss가 가장 큰 상위 10% 영역은 noisy label일 수 있어 무시한다. Labeled (synthetic) 이미지에는 MiDaS가 제안한 두 loss를 쓴다. 첫째는 V1과 같은 scale- and shift-invariant loss이다.

$$
\mathcal{L}_{ssi} = \frac{1}{HW}\sum_{i=1}^{HW}\lvert\hat{d}_i^* - \hat{d}_i\rvert
$$

$$d_i^*$$는 예측, $$d_i$$는 GT이고, $$\hat{d}$$는 median과 평균 절대편차로 scale과 shift를 맞춘 값이다 (V1 참고). 둘째는 gradient matching loss이다.

$$
\mathcal{L}_{gm} = \frac{1}{HW}\sum_{k=1}^{K}\sum_{i=1}^{HW}\left(\lvert\nabla_x R_i^k\rvert + \lvert\nabla_y R_i^k\rvert\right), \qquad R_i = \hat{d}_i^* - \hat{d}_i
$$

$$R^k$$는 $$k$$번째 해상도에서의 차이 map이다 (MiDaS는 $$K=4$$). 쉽게 말해, 예측 depth가 바뀌는 곳 (경계)이 GT가 바뀌는 곳과 같은 위치에서 같은 크기로 바뀌도록 만드는 loss이다.

MiDaS에서는 real 데이터의 GT가 sparse하고 거칠어 $$\mathcal{L}_{gm}$$의 효과가 뚜렷하지 않았다. 하지만 GT가 정밀한 synthetic 이미지에서는 **depth sharpness에 크게 도움**이 되어, 가중치를 0.5에서 4.0으로 키울수록 경계가 날카로워졌다. 최종적으로 수치와 sharpness를 절충해 두 loss의 비율을 1:2로 둔다.

$$
\mathcal{L} = \mathcal{L}_{ssi} + 2\,\mathcal{L}_{gm}
$$

Pseudo-labeled 이미지에는 V1처럼 feature alignment loss를 더해 사전학습된 DINOv2 encoder의 semantic 정보를 유지한다.

### 5. A New Evaluation Benchmark: DA-2K

학습용 real 데이터뿐 아니라 널리 쓰이는 test benchmark도 noisy하다.

![Noisy labels in the NYU-D test set](fig8.webp)
_출처: 원 논문 Figure 8_

NYU-D는 전용 depth sensor를 쓰는데도 거울과 얇은 구조의 depth가 틀려 있다. 이런 noise 때문에 **더 좋은 모델이 오히려 낮은 점수**를 받는다. 기존 benchmark의 문제는 세 가지다.

- **Label noise**: 위처럼 GT 자체가 틀린 곳이 많다.
- **Limited diversity**: 대부분 한 종류의 scene만 담는다 (NYU-D는 몇 개의 실내 방, KITTI는 몇 개의 도로).
- **Low resolution**: 대부분 500×500 안팎인데, 요즘 카메라 이미지는 1000×2000 정도라 결론이 고해상도에서도 성립하는지 알 수 없다.

![DA-2K annotation pipeline and scenarios](fig9.webp)
_출처: 원 논문 Figure 9_

따라서, 적절한 평가 벤치마크를 새로 제안한다. 벤치마크 구축은 2단계로 볼 수 있다.

- **Pipeline 1**: SAM의 mask를 만드는 key point 중 두 개를 무작위로 뽑고, 네 모델 (V1, Marigold, GeoWizard, V2)이 상대 depth를 투표한다. 의견이 갈리면 사람이 판정한다.(직접 annotation)
- **Pipeline 2**: 모든 모델이 똑같이 틀리는 어려운 쌍은 투표로 걸러지지 않으므로, 이미지를 직접 보고 어려운 쌍을 사람이 고른다.

다양성을 위해 MDE의 주요 응용 scenario 8개 (indoor, outdoor, non-real, transparent/reflective, adverse style, aerial, underwater, object)를 정하고, GPT-4로 만든 keyword로 Flickr에서 이미지를 받았다. 총 1K장, 2K개의 픽셀 쌍이다.

DA-2K가 기존 benchmark를 대체하려는 것은 아니다. 정확한 sparse depth는 scene reconstruction에 필요한 정밀한 dense depth와 거리가 멀다. 다만 정밀한 dense depth의 전제 조건이므로, 다양한 scene과 정밀도를 갖춘 보완 benchmark로 쓸 수 있다.

---

## Results

- 자세한 실험 설정은 논문 참고

### 1. Zero-Shot Relative Depth Estimation

![Zero-shot relative depth estimation](table2.webp)
_출처: 원 논문 Table 2_

- MiDaS보다는 좋지만 **V1과는 비슷**하고, 두 데이터셋에서는 V1보다 약간 낮다.
- 하지만 V2가 노린 fine detail, complex layout과 transparent object에 대한 robustness는 이런 benchmark에 제대로 반영되지 않는다. 앞서 말한 평가 데이터셋의 오류 때문.

### 2. Performance on DA-2K

![Performance on DA-2K](table3.webp)
_출처: 원 논문 Table 3_

- 제안한 벤치마크에서의 평가.
- **가장 작은 ViT-S 모델 (95.3%)도** 무거운 SD 기반 모델 Marigold (86.8%), GeoWizard (88.1%)와 V1 (88.5%)보다 훨씬 좋다. 가장 큰 ViT-G 모델은 97.4%로 Marigold보다 10.6%p 높다.

### 3. Fine-tuned to Metric Depth Estimation

![In-domain metric depth estimation on NYU-D and KITTI](table4.webp)
_출처: 원 논문 Table 4_

V1처럼 ZoeDepth pipeline에서 MiDaS encoder를 V2 encoder로 바꾸고, 학습과 테스트 domain이 같은 in-domain metric depth로 fine-tuning한다.

- NYU-D와 KITTI 모두에서 기존 방법보다 크게 좋다. ViT-L은 NYU-D에서 δ1 0.984, KITTI에서 AbsRel 0.045.
- **가장 가벼운 ViT-S 모델도** ViT-L 기반 ZoeDepth보다 좋다.
- 다만 NYU-D, KITTI로 학습한 metric 모델은 학습셋의 noise 때문에 fine detail이 부족하고 transparent object에 약하다. 그래서 실제 응용용으로는 Hypersim (실내)과 Virtual KITTI (실외) synthetic 데이터로 fine-tuning한 metric depth 모델을 따로 공개한다.

### 4. Ablation Studies

![Importance of pseudo-labeled real images](table5.webp)
_출처: 원 논문 Table 5_

**Pseudo-labeled real 이미지의 중요성.** Synthetic 이미지 ($$\mathcal{D}^l$$)만 쓸 때보다 pseudo-labeled real 이미지 ($$\mathcal{D}^u$$)를 더하면 크게 좋아진다 (ViT-S의 DA-2K 89.8% → 94.1%). V1과 달리 student 학습에서 synthetic 이미지를 아예 빼 봤더니, 작은 모델 (ViT-S, ViT-B)은 오히려 약간 더 좋았다 (ViT-S 95.3%). 그래서 student는 pseudo-labeled 이미지로만 학습한다.

![Synthetic-only vs pseudo-labeled-only training](fig16.webp)
_출처: 원 논문 Figure 16_

DINOv2-S 기반 모델을 labeled synthetic 이미지로만 학습한 경우 (가운데)와 pseudo-labeled real 이미지로만 학습한 경우 (오른쪽). Pseudo-labeled real 이미지를 쓰면 **robustness가 크게 좋아진다**.

![Pseudo label vs manual label on DIML](table6.webp)
_출처: 원 논문 Table 6_

**Pseudo label vs. manual label.** DIML 데이터셋의 real 이미지를 원래의 수동 label로 학습한 경우와 teacher의 pseudo label로 학습한 경우를 비교했다. 모든 데이터셋에서 pseudo label 쪽이 확실히 좋다. Pseudo label의 품질이 높고, 기존 labeled real 데이터에 noise가 많다는 뜻이다.

---

## Conclusion

본 논문은 MDE를 위한 더 강력한 foundation model Depth Anything V2를 제안함. 화려한 기법 대신 데이터 설계를 다시 보는 것으로 다음을 달성.

(1) Robust하면서 fine-grained한 depth 예측.

(2) 25M~1.3B의 다양한 모델 크기로 넓은 응용 지원.

(3) Downstream task로 쉽게 fine-tuning할 수 있는 좋은 초기값.

또 기존 test set의 낮은 diversity와 많은 noise를 보완하기 위해, 다양한 고해상도 이미지와 정밀한 sparse depth label을 갖춘 benchmark DA-2K를 만들었다.

**Limitation.** 62M장의 unlabeled 이미지로 학습하는 계산 부담이 매우 크다. 또 현재 synthetic 학습셋이 충분히 다양하지 않아, 더 많은 출처에서 synthetic 이미지를 모아 더 강한 teacher를 만드는 것이 다음 과제이다.
