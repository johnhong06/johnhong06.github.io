---
title: "Spatial Forcing: Implicit Spatial Representation Alignment for Vision-language-action Model (ICLR 2026)"
date: 2026-11-20 15:30:00 +0900
categories: [논문리뷰]
tags: [vision-language-action model, robot manipulation, 3d foundation model, representation alignment]
math: true
description: "2D 이미지로만 사전학습된 VLA는 3D 공간을 이해하는 데 어려움이 있다. 3D 정보로 학습하는 것 또한 단점이 존재한다. 따라서, 중간 layer visual token을 3D foundation model(VGGT)의 feature에 정렬하는 loss 하나를 더해, depth 입력 없이도 공간 이해를 갖추게 한다. 추론 때는 아무것도 추가되지 않는다."
media_subpath: /assets/img/posts/spatial-forcing/
---

**ICLR 2026.** [Paper](https://arxiv.org/abs/2510.12276) [Project Page](https://spatial-forcing.github.io/) [Github](https://github.com/OpenHelix-Team/Spatial-Forcing)<br>
Fuhao Li, Wenxuan Song, Han Zhao, Jingbo Wang, Pengxiang Ding, Donglin Wang, Long Zeng, Haoang Li<br>
HKUST (Guangzhou) \| Tsinghua University \| Westlake University \| Zhejiang University \| South China University of Technology<br>
17 Oct 2025 (arXiv v2)

핵심 한 줄: 2D 이미지로만 사전학습된 VLA는 3D 공간을 이해하는 데 어려움이 있다. 3D 정보로 학습하는 것 또한 단점이 존재한다. 따라서, 중간 layer visual token을 3D foundation model(VGGT)의 feature에 정렬하는 loss 하나를 더해, depth 입력 없이도 공간 이해를 갖추게 한다. 추론 때는 아무것도 추가되지 않는다.

![Spatial Forcing overview, performance, and depth probing](fig1.webp)
_출처: 원 논문 Figure 1_

대부분의 VLA는 2D 이미지로만 사전학습된 VLM 위에 만들어져 3D 공간 인식이 약하다. Depth map·point cloud를 입력으로 넣는 방법은 센서 품질과 데이터 문제가 있고, depth estimator로 3D를 추정하는 방법은 estimator 성능에 묶인다. 저자들은 학습 중에만 VLA 중간 layer의 visual token을 VGGT feature에 맞추는 Spatial Forcing(SF)을 제시한다. 그 결과, LIBERO 평균 98.5%로 depth를 입력으로 쓰는 3D VLA보다도 높고, 같은 성공률에 3.8배 빨리 도달하며 적은 데이터로도 잘 학습된다.

---

## Introduction

VLA는 VLM에 action tokenization이나 action expert를 붙여 언어 지시를 로봇 action으로 바꾼다. 하지만 VLM backbone은 2D 이미지로만 사전학습되어 정확한 공간 인식이 부족하고, 3D 물리 세계에서 정밀한 조작을 하기 어렵다.

이를 보완하려는 3D VLA는 크게 두 갈래다.

![Comparison of 3D VLA approaches](fig2.webp)
_출처: 원 논문 Figure 2_

1. 명시적 3D 입력 (Fig. 2a): depth camera·lidar로 얻은 depth map이나 point cloud를 encoding해 VLM backbone이나 action head에 넣는다 (PointVLA, GeoVLA, 3D-CAVLA 등).
2. 2D 이미지에서 3D 추정 (Fig. 2b): depth estimator 같은 별도 3D 모델로 3D 정보를 추정해 넣는다 (SpatialVLA, EVO-0).

\[문제점1\]<br>
(1) 센서로 얻은 depth·point cloud는 품질이 낮고 부정확하다.

(2) 로봇마다 센서 종류·위치·calibration이 달라 데이터가 이질적이다.

(3) 기존 대규모 로봇 데이터셋(Open X-Embodiment 등)의 일부에는 depth가 아예 없어 scaling이 어렵다.

\[문제점2\]<br>
2D 이미지에서 3D를 추정하는 방법은 depth estimator의 성능이 곧 상한이라 policy가 sub-optimal해진다.

\[문제점3\]<br>
VLA의 visual embedding에는 공간 정보가 부족하다. Frozen VLA(OpenVLA-OFT)의 visual embedding 위에 DPT head만 학습해 depth를 예측시키면 의미 있는 공간 구조가 나오지 않는다 (Fig. 1c). 그런데 기존 3D VLA는 입력을 3D로 풍부하게 하는 데만 집중했고, action token을 만드는 중간 표현인 visual embedding 자체는 다루지 않았다.

\[해결책\]<br>
입력은 RGB 그대로 두고, 학습 때만 VLA 중간 layer의 visual token을 사전학습된 3D foundation model(VGGT)의 feature에 정렬한다. 3D 센서도 depth estimator도 필요 없고, VLA가 공간 정보를 내부 표현에 스스로 담도록 '강제'한다.

Contribution은 다음과 같다.

- Depth probing으로 VLA의 visual embedding에 공간 정보가 부족하다는 것을 보인다.
- VLA의 visual embedding을 외부 3D representation에 정렬하는 간단한 방법 Spatial Forcing을 제안한다.
- 시뮬레이션(LIBERO·RoboTwin)과 실제 로봇에서 성능, 학습 속도, 데이터 효율을 모두 높인다.

---

## Method

### 1. Preliminaries: VLA

$$
x_t^{\mathcal{A}} \sim p_\theta\left(x_t^{\mathcal{A}} \mid \{x_i^{\mathcal{V}}\}_{i=1}^{N},\, \{x_j^{\mathcal{L}}\}_{j=1}^{M},\, x_{<t}^{\mathcal{A}}\right), \qquad \mathcal{L}_{\text{action}} = \mathcal{L}\left[\mathcal{G}\left(\{x_t^{\mathcal{A}}\}_{t=1}^{K}\right),\, A_{gt}\right]
$$

- Multi-view 이미지는 DINOv2·SigLIP 같은 visual encoder를 거쳐 visual token N개, 지시문은 linguistic token M개가 되고, VLA는 이를 조건으로 action token K개를 만든다.
- \\(\mathcal{G}\\)는 학습되는 action expert(2-layer MLP, flow-matching head 등), $$\mathcal{L}$$은 L1·L2·cross-entropy 같은 학습 loss다.
- Causal attention이라 action token은 앞의 visual token을 보고 만들어진다. 즉 visual token이 장면을 담는 중간 표현이고, 여기를 직접 supervise할 수 있다.

### 2. VGGT (Visual Geometry Grounded Transformer)

- 여러 장의 2D 이미지에서 camera parameter, point map, depth map, 3D point track을 한 번에 내는 feed-forward model이다.
- 저자들은 head 이전 backbone의 latent representation이 이미 풍부한 공간 정보를 담고 있어 3D supervision signal로 충분하다고 본다. 따라서 저자들은 VGGT의 풍부한 feature를 alignment에 활용한다.

### 3. Motivation: Depth probing

![Depth probing with and without alignment](fig3.webp)
_출처: 원 논문 Figure 3_

- VLA의 파라미터를 모두 얼리고 DPT head만 학습해 visual embedding에서 depth map을 예측시킨다 (linear probing과 같은 발상). Embedding에 공간 정보가 얼마나 담겼는지 보는 실험이다.
- 2D로만 학습한 embedding(w/o Alignment)에서는 의미 있는 depth 구조가 나오지 않고, SF로 정렬한 embedding(w/ Alignment)에서는 GT에 가까운 depth가 나온다. LIBERO 시뮬레이션과 실제 로봇 모두 같은 경향이다.

요약하면, 간단하게 깊이 정보가 있는지 확인하는 방법으로 Depth Probing을 적용. 그 결과, VLA 내부에 정보가 부족함을 확인하였다.

### 4. Spatial Forcing

$$
\mathcal{L}_{\text{align}} = -\frac{1}{N} \sum_{i=1}^{N} S\left[\text{MLP} \cdot \Gamma\left(x_i^{\mathcal{V}}\right),\; f_i^{3D}(\mathcal{I}) + E\right]
$$

- Multi-view 이미지 묶음 $$\mathcal{I}$$를 frozen VGGT $$f^{3D}$$에 넣어 pixel 단위 spatial representation을 얻고, 위치 순서를 보존하려고 positional embedding $$E$$를 더해 target으로 쓴다.
- VLA의 i번째 visual token $$x_i^{\mathcal{V}}$$는 batch normalization $$\Gamma$$와 2-layer MLP를 거쳐 VGGT feature 차원으로 바뀐다.
- \\(S\\)는 cosine similarity다. 같은 위치의 VLA token과 VGGT feature가 같은 방향을 가리키도록 만든다. VGGT patch feature는 VLA patch 격자 크기로 bilinear 보간해 1:1로 짝짓는다.
- 어느 layer를 정렬할지도 중요하다. 꽤 깊지만 마지막은 아닌 layer(32개 중 24번째)가 가장 좋다. 마지막 몇 layer는 vision과 language가 modality-agnostic 공간으로 섞여 vision 고유 feature가 줄어들기 때문이다.

$$
\mathcal{L}_{SF} = \mathcal{L}_{\text{action}} + \alpha \mathcal{L}_{\text{align}}
$$

- 기본값은 $$\alpha = 0.5$$다 (Appendix Table 3).
- 추론 때는 VGGT도 MLP도 쓰지 않는다. 일반 VLA와 구조·계산량이 같다.

핵심: **학습 때만 VLA 중간 layer의 visual token을 VGGT feature에 cosine similarity로 맞추는 보조 loss를 더한다.**

---

## Results

- 자세한 실험 설정은 논문 참고

### 1. LIBERO

4개 suite(Spatial·Object·Goal·Long), suite마다 10개 task. OpenVLA-OFT를 base로 primary·wrist 카메라를 쓰고, 8×H100에서 150k iteration 학습했다.

![LIBERO results](table1.webp)
_출처: 원 논문 Table 1_

**SF는 RGB만 쓰고도 모든 2D VLA를 앞서고, depth·point cloud를 입력으로 쓰는 3D VLA와 비슷하거나 더 높다.**

- 평균 SR은 SF 98.5, base인 OpenVLA-OFT 97.1, 3D 센서 입력을 쓰는 GeoVLA 97.7·3D-CAVLA 98.1이다.

### 2. RoboTwin 2.0

양팔 benchmark. Easy는 in-domain layout, Hard는 domain randomization(clutter, 배경 texture, 조명, 테이블 높이)이다. π0를 base로 LoRA, 1×H100에서 30k iteration 학습했다.

![RoboTwin 2.0 results](fig4.webp)
_출처: 원 논문 Figure 4_

**SF는 평균 SR이 가장 높고, 특히 domain randomization이 들어간 Hard에서 base π0보다 크게 오른다.** 배경·조명 같은 shortcut 대신 물체 위치와 상대적 공간 관계를 본다는 해석이다.

- 평균 SR(π0 → SF)은 그래프 기준 Easy 약 49 → 55, Hard 약 22 → 31이다.

### 3. Ablation: 정렬 target

이하 ablation은 모두 1×H100에서 OpenVLA-OFT를 LIBERO로 학습했다. 첫 행(✗)은 SF 없는 base로 평균 92.7이다 (8 GPU로 학습한 Table 1의 97.1보다 낮다).

![Ablation on alignment target](table2a.webp)
_출처: 원 논문 Table 2 일부_

**어떤 target에 정렬해도 base보다 좋아지고, 3D 정보를 가진 VGGT가 가장 좋다. Positional embedding을 빼면 Long에서 크게 떨어진다.**

- 평균 SR은 SigLIP 94.0, DINOv2 94.1, VGGT 96.9이고, PE를 빼면 Long이 94.2 → 84.4로 떨어진다.

### 4. Ablation: 정렬할 layer

![Ablation on aligned layer](table2b.webp)
_출처: 원 논문 Table 2 일부_

**24번째 layer를 정렬할 때 가장 좋다.** 얕은 layer를 맞추면 이후 layer를 거치며 공간 정보가 사라지고, 마지막 layer는 vision 고유 feature가 줄어 정렬이 잘 되지 않는다.

- 평균 SR은 1번째 94.6, 8번째 95.7, 16번째 93.8, 24번째 96.9, 32번째 94.8이다.

### 5. 학습 효율과 데이터 효율

(b)는 전체 데이터에서 1%·5%·33%를 균일하게 뽑아 학습했다 (scheduler는 cosine annealing).

![Training and data efficiency](fig5ab.webp){: w="550" }
_출처: 원 논문 Figure 5(a)(b)_

**SF는 같은 성공률에 3.8배 적은 iteration으로 도달하고, 5.9배 적은 데이터로 같은 성공률을 낸다.**

- 그래프는 Table 2의 LIBERO-Spatial 열 값이다. SF는 5k iteration에 93.8로 base가 20k에서 내는 수준(약 94)에 닿고, 데이터 5%만으로 4개 suite 평균 75.8%를 낸다.

### 6. t-SNE

![t-SNE of VLA and VGGT features](fig5c.webp){: w="240" }
_출처: 원 논문 Figure 5(c)_

**정렬 후 VLA feature 분포는 target(VGGT)과 모양이 거의 같아지지만, 중심은 target과 겹치지 않는다.**

- 모양이 같다는 것은 feature 사이의 상대적 구조를 배웠다는 뜻이고, 중심이 다르다는 것은 target을 그대로 복사하지 않고 visual 표현 고유의 정보를 유지했다는 뜻이라고 저자들은 해석한다 (Appendix B).

### 7. Ablation: α (Appendix A)

![Ablation on alignment loss weight](table3.webp){: w="430" }
_출처: 원 논문 Table 3_

**정렬 loss는 작은 가중치만 줘도 크게 좋아지지만, 너무 크면 action 학습을 방해한다.**

- α = 0에서 73.2, 0.5에서 93.6으로 가장 높고, 12.5에서는 81.2로 떨어진다. (α = 0은 정렬이 없는 base인데 Table 2의 base 92.7과 달라 다른 설정으로 잰 것으로 보인다. 논문에 설정 설명은 없음.)

### 8. 실제 로봇

양팔 AgileX 플랫폼(6-DoF Piper 팔 + 1-DoF gripper)에 primary 카메라 1개와 wrist 카메라 2개. 한 팔 task는 시연 40개, 양팔 task는 20개로 학습했고, 한 팔 task는 변형 4종 × 10 trial, 양팔은 20 trial로 평가했다.

![Real-world experiments](fig6.webp)
_출처: 원 논문 Figure 6_

**적은 시연만으로 4개 task 모두에서 SF가 base보다 높고, 조명이 바뀌는 투명 컵 쌓기에서 차이가 가장 크다.**

- SR(w/o SF → w/ SF)은 Stack Glass Cups 15.0 → 62.5, Grasp Right-side Vegetable 10.0 → 47.5, Place Green Block 67.5 → 85.0, Lift Pot(양팔) 30.0 → 42.5이다.

---

## Conclusion

VLA는 2D 이미지로만 사전학습된 VLM 위에 만들어져 visual embedding에 공간 정보가 부족하다는 것을 depth probing으로 보였다. Spatial Forcing은 학습 때만 VLA 중간 layer의 visual token을 VGGT feature에 정렬해, 3D 센서나 depth estimator 없이 공간 이해를 VLA 안에 심는다. 그 결과 LIBERO·RoboTwin에서 2D·3D VLA를 앞서고, 학습은 최대 3.8배 빠르며 데이터도 덜 든다. 추론 구조는 원래 VLA와 같아 다른 VLA에도 쉽게 붙일 수 있다.
