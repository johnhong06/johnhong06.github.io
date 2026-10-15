---
title: "Depth Anything 3: Recovering the Visual Space from Any Views (ICLR 2026)"
date: 2026-10-15 12:20:00 +0900
categories: [논문리뷰]
tags: [multi-view geometry, camera pose estimation, monocular depth estimation, 3d gaussian splatting]
math: true
description: "입력 이미지가 몇 장이든 3D 공간을 복원하는 Vanilla transformer 하나 (DINOv2)와 Depth + Ray 예측이면 충분하다."
media_subpath: /assets/img/posts/depth-anything-3/
---

**ICLR 2026.** [Paper](https://proceedings.iclr.cc/paper_files/paper/2026/hash/e4cd50120b6d7e8daff1749d6bbaa889-Abstract-Conference.html) [Project Page](https://depth-anything-3.github.io) [Github](https://github.com/ByteDance-Seed/Depth-Anything-3)<br>
Haotong Lin, Sili Chen, Jun Hao Liew, Donny Y. Chen, Zhenyu Li, Yang Zhao, Sida Peng, Hengkai Guo, Xiaowei Zhou, Guang Shi, Jiashi Feng, Bingyi Kang<br>
ByteDance Seed \| Zhejiang University \| ByteDance Intelligent Creation<br>
13 Nov 2025 (arXiv v1), 표와 수치는 ICLR 2026 최종본 기준

핵심 한 줄: 입력 이미지가 몇 장이든 3D 공간을 복원하는 Vanilla transformer 하나 (DINOv2)와 Depth + Ray 예측이면 충분하다.

![Depth Anything 3 overview](fig1.webp){: w="640" }
_출처: 원 논문 Figure 1_

이미지 1장, 여러 장, 영상 등 몇 장이 들어오든, camera pose가 있든 없든 각 이미지의 depth map과 ray map을 예측하고, 둘을 합쳐 일관된 point cloud를 만든다. 구조를 바꾸지 않은 vanilla DINOv2에 토큰 재배열로 cross-view attention을 넣고, Dual-DPT head로 depth와 ray를 함께 출력한다. 합성 데이터로 학습한 teacher의 pseudo label로 실제 데이터를 학습하며, 새로 만든 visual geometry benchmark에서 기존 SOTA인 VGGT보다 **camera pose 정확도는 평균 35.7%, geometry 정확도는 23.6% 높다**. 단안 depth에서도 DA2를 앞선다.

---

## Introduction

본 논문은 기존 3D task 정의에서 한 발 물러나, 이미지 1장이든 여러 view든 영상이든 **임의의 시각 입력에서 3D 구조를 복원**한다는 근본적인 목표로 돌아간다. 복잡한 구조 설계 대신 minimal modeling을 추구하며, 두 질문에서 출발한다.

- **Minimal set of prediction targets**: 최소한의 예측 대상이 있는가, 아니면 여러 3D task를 함께 모델링해야 하는가? → Depth + Ray면 충분하다.
- **Single plain transformer**: 평범한 transformer 하나로 충분한가? → 충분하다. Vanilla ViT (DINOv2)를 그대로 쓴다.

1. N장의 입력 이미지마다 pixel-aligned **depth map**과 **ray map**을 출력하는 dense prediction 문제로 정의한다.
2. 사전학습 ViT (DINOv2)를 backbone으로 쓰고, 일부 layer에서 토큰을 재배열해 view끼리 정보를 주고받게 한다 (**input-adaptive** cross-view self-attention).
3. Dual-DPT head가 같은 feature에서 depth와 ray를 함께 출력한다.
4. Camera pose를 알면 camera encoder로 넣어 줄 수 있다.
5. 합성 데이터로 학습한 teacher의 pseudo depth로 출처가 다양한 학습 데이터를 하나로 묶는다.

\[문제점1\]<br>
SfM, MVS, SLAM, MDE는 사실상 입력 view 개수 하나만 다른데도, **task마다 전용 모델을 따로 만드는 것**이 주류였다.

\[해결책1\] - **<u>Input Adaptive 하다!</u>**<br>
Task 구분 없이 몇 장의 이미지든, pose가 있든 없든 하나의 모델이 depth와 pose를 함께 추정한다. 이미지가 1장이면 그대로 monocular depth estimation이 된다.

\[문제점2\]<br>
VGGT, DUSt3R 같은 통합 시도가 있었지만 복잡한 전용 구조에 기대고 여러 task를 scratch부터 함께 학습해, **대규모 사전학습 가중치의 장점을 제대로 살리지 못한다**.

\[해결책2\]<br>
구조를 바꾸지 않은 평범한 transformer (vanilla DINOv2) 하나를 backbone으로 쓰고, **토큰 재배열만으로 cross-view attention을 구현**한다. 사전학습 가중치와 그 scaling 특성을 그대로 물려받는다.

\[문제점3\]<br>
학습 데이터는 depth camera 촬영, 3D reconstruction, 합성 데이터 등 출처가 다양하고, **실제 데이터의 depth는 noisy하거나 sparse하다**.

\[해결책3\]<br>
**합성 데이터로만 강한 monocular depth teacher를 학습**해 모든 Real image의 dense하고 정밀한 pseudo depth를 만든다. 이 pseudo depth를 원래의 sparse·noisy depth에 맞춰 정렬해, 기하 정확도는 지키면서 label의 detail과 completeness를 높인다.

Contribution은 다음과 같다.

- 평범한 transformer 하나와 depth-ray 예측만으로 any-view geometry를 통합할 수 있음을 보인다.
- 합성 데이터 teacher로 출처가 다양한 실제 데이터를 묶는 teacher-student 학습을 제시한다.
- Camera pose, geometry, rendering을 평가하는 visual geometry benchmark를 제안한다.
- 20개 설정 중 18개에서 SOTA를 달성하고, 단안 depth에서도 DA2를 앞선다. DPT head 하나만 더해 feed-forward NVS에서도 가장 좋은 backbone이 된다.

---

## Related Work

**Multi-view visual geometry estimation.** 전통적인 시스템은 복원을 여러 단계로 나눈다.

1. Feature detection과 matching
2. Robust한 relative pose estimation (epipolar geometry)
3. SfM과 bundle adjustment
4. Dense multi-view stereo로 view별 depth와 point cloud 생성

이 방식에는 두 가지 약점이 있다.

- **모듈형**: 단계별로 진행되어 앞 단계의 오차가 누적된다.
- **Feature 의존성**: texture가 없거나, 반사가 있거나, 시점 변화가 크면 대응점 찾기가 깨진다.

DUSt3R가 transformer로 두 view의 point map을 직접 예측해 depth와 relative pose를 feed-forward로 구하면서 전환점이 되었고, 이후 multi-view, video 입력, camera parameter 주입 등으로 확장됐다. 그중 VGGT는 대규모 학습, multi-stage 구조, 중복된 설계로 정확도를 크게 끌어올렸다. 본 논문은 반대로 **단순한 transformer 하나에 집중**한다.

**Monocular depth estimation.** 초기에는 실내나 도로 같은 단일 도메인에서만 지도학습해 새로운 환경으로 일반화하기 어려웠다. 이후 MiDaS, Metric3D, Depth Anything V1·V2처럼 대규모 데이터셋과 강한 사전학습 모델로 일반화 성능이 크게 올랐다. DA3는 MDE 전용 모델이 아니라 depth와 camera pose를 함께 다루는 **unified visual geometry estimation 모델**이지만, MDE 성능도 기존 모델과 같거나 더 좋다.

---

## Depth Anything 3

![Pipeline of Depth Anything 3](fig2.webp)
_출처: 원 논문 Figure 2_

전체 pipeline. vanilla DINOv2 하나에 input-adaptive cross-view self-attention을 넣고, Dual-DPT head로 depth map과 ray map을 예측한다. Camera parameter가 있으면 camera token으로 encoding해 patch token과 함께 모든 attention에 참여시킨다.

### 1. Formulation

입력은 $$\mathcal{I} = \{\mathbf{I}_i\}_{i=1}^{N_v}$$, $$\mathbf{I}_i \in \mathbb{R}^{H\times W\times 3}$$이다. $$N_v = 1$$이면 단안 이미지, $$N_v > 1$$이면 영상이나 multi-view 집합이다. 각 이미지는 depth $$\mathbf{D}_i \in \mathbb{R}^{H\times W}$$, camera pose $$[\mathbf{R}_i \mid \mathbf{t}_i]$$, intrinsics $$\mathbf{K}_i$$를 가진다. Camera는 translation $$\mathbf{t}_i \in \mathbb{R}^3$$, rotation quaternion $$\mathbf{q}_i \in \mathbb{R}^4$$, FOV $$\mathbf{f}_i \in \mathbb{R}^2$$를 이어 붙인 $$\mathbf{v}_i \in \mathbb{R}^9$$로도 나타낼 수 있다. 픽셀 $$\mathbf{p} = (u, v, 1)^\top$$은 다음 식으로 3D 점 $$\mathbf{P}$$가 된다.

$$
\mathbf{P} = \mathbf{R}_i\big(\mathbf{D}_i(u,v)\,\mathbf{K}_i^{-1}\mathbf{p}\big) + \mathbf{t}_i
$$

쉽게 말해, 2D 픽셀을 depth만큼 camera 앞으로 보낸 뒤 회전·이동해 3D 월드 좌표로 옮기는 식이다.

**① Depth-ray representation.** 유효한 회전 행렬 $$\mathbf{R}_i$$를 직접 예측하기는 직교성 제약 (orthogonality constraint) 때문에 어렵다. 그래서 camera pose를 입력 이미지, depth map과 정렬된 **픽셀별 ray map으로 간접 표현**한다. 각 픽셀의 ray $$\mathbf{r} \in \mathbb{R}^6$$은 원점 $$\mathbf{t} \in \mathbb{R}^3$$ (3채널)과 방향 $$\mathbf{d} \in \mathbb{R}^3$$ (3채널)으로 이루어진다: $$\mathbf{r} = (\mathbf{t}, \mathbf{d})$$. 방향은 픽셀을 camera 좌표로 역투영한 뒤 월드 좌표로 회전한 것이다.

$$
\mathbf{d} = \mathbf{R}\mathbf{K}^{-1}\mathbf{p}
$$

모든 픽셀의 ray를 모은 것이 ray map $$\mathbf{M} \in \mathbb{R}^{H\times W\times 6}$$이다. $$\mathbf{d}$$를 정규화하지 않아 크기에 투영 scale이 남아 있으므로, 3D 점은 단순히 다음과 같다.

$$
\mathbf{P} = \mathbf{t} + \mathbf{D}(u,v)\cdot\mathbf{d}
$$

<u>결국 3D 복원에 필요한 최소 정보는 Ray, Depth라는 것이다. 이 2개를 알면 3D 복원이 가능하다는 것이다.</u>

**② Deriving camera parameters from the ray map.** Ray map에서 **camera의 위치, 회전, 렌즈 값을 되찾을 수 있다**.

1. 위치: camera 중심 $$\mathbf{t}_c$$는 픽셀별 ray 원점의 평균이다.

   $$
   \mathbf{t}_c = \frac{1}{H\times W}\sum_{h=1}^{H}\sum_{w=1}^{W}\mathbf{M}(h, w, :3)
   $$

2. 회전과 렌즈: intrinsics가 단위행렬인 identity camera의 ray $$\mathbf{d}_I = \mathbf{p}$$와 실제 camera의 ray $$\mathbf{d}_{cam} = \mathbf{K}\mathbf{R}\mathbf{d}_I$$ 사이에는 homography $$\mathbf{H} = \mathbf{K}\mathbf{R}$$ 관계가 있다. 이 $$\mathbf{H}$$를 DLT (Direct Linear Transform)로 푼다.

   $$
   \mathbf{H}^* = \arg\min_{\lVert\mathbf{H}\rVert=1}\sum_{h=1}^{H}\sum_{w=1}^{W}\left\lVert \mathbf{H}\mathbf{p}_{h,w} \times \mathbf{M}(h, w, 3:)\right\rVert
   $$

3. 분리: $$\mathbf{K}$$는 상삼각 행렬, $$\mathbf{R}$$은 직교 행렬이므로 $$\mathbf{H}^*$$를 RQ 분해하면 둘을 유일하게 구할 수 있다.

쉽게 말해, 예측한 ray 방향들을 보고 단위 camera의 ray를 어떤 KR로 바꾸면 이 방향들이 나오는지 푼 다음, 그 KR을 렌즈 (K)와 회전 (R)으로 쪼개는 것이다.

**③ Minimal prediction targets.** 왜 Depth + Ray만 있으면 될까?

- DUSt3R처럼 point map만 예측하면 depth와 camera 정보가 한 표현에 얽혀 있어, depth를 따로 예측하는 것보다 geometry 추정에 불리하다.
- 그래서 VGGT 등은 point map 옆에 depth head를 추가로 두는데, 이러면 예측 대상이 중복된다.
- Depth-ray는 장면 구조와 camera 움직임을 모두 담으면서 **서로 분리된 (disentangled) 최소한의 target**이다 (Table 5에서 확인).

\[문제점\]<br>
Ray map에서 pose를 복원하려면 평균 + DLT + RQ 분해를 거쳐야 해서 **inference 때 계산이 무겁다**.

\[해결책\]<br>
**가벼운 camera head** $$\mathcal{D}_C$$를 추가한다. View마다 camera token 1개만 받아 FOV $$\mathbf{f}$$, quaternion $$\mathbf{q}$$, translation $$\mathbf{t}$$를 바로 출력하므로 추가 비용은 backbone의 약 0.1%이다. Ray map에서 푸는 것 (8.60ms)보다 18.7배 빠르고 (0.46ms), 정확도는 ray 기반이 약간 더 높다 (DA3-Giant 평균 Auc3 68.0 vs 63.8).

### 2. Architecture

**① Single transformer backbone.** 대규모 단안 이미지로 사전학습된 $$L$$개 block의 ViT (DINOv2)를 그대로 쓴다. **구조 변경 없이 입력 토큰을 재배열하는 것만으로** cross-view reasoning을 한다.

- 앞쪽 $$L_s$$개 layer는 각 이미지 안에서만 self-attention을 한다 (within-view).
- 뒤쪽 $$L_g$$개 layer는 cross-view와 within-view attention을 번갈아 한다. Tensor 순서만 바꿔 모든 view의 토큰을 함께 본다.
- \\(L_s : L_g = 2 : 1\\)이 성능과 효율의 균형이 가장 좋다 (Table 6).
- Input-adaptive: 이미지가 1장이면 추가 비용 없이 그대로 monocular depth estimation이 된다.

**② Camera condition injection.** 각 view 앞에 camera token $$\mathbf{c}_i$$를 붙인다.

- Camera parameter를 알면 MLP $$\mathcal{E}_c$$로 만든 $$\mathbf{c}_i = \mathcal{E}_c(\mathbf{f}_i, \mathbf{q}_i, \mathbf{t}_i)$$를 쓴다.
- 모르면 모든 view가 공유하는 learnable token $$\mathbf{c}_l$$을 쓴다.
- Camera token은 patch token과 함께 모든 attention에 참여해 camera 맥락을 제공한다.

**③ Dual-DPT head.**

![Dual-DPT head](fig3.webp){: w="400" }
_출처: 원 논문 Figure 3_

- Reassemble (공유): backbone 토큰을 feature map으로 조립하는 모듈을 depth와 ray가 같이 쓴다. → **두 출력이 정렬**되고 중복된 중간 표현이 없다.
- Fusion (분리): depth용, ray용 fusion layer와 output layer를 따로 두어 각각 depth map과 ray map을 만든다.
- Depth에 대한 confidence map도 함께 출력한다.

### 3. Teacher-Student Learning

**① Constructing the teacher model.**

![Poor quality real-world datasets](fig7.webp){: w="400" }
_출처: 원 논문 Figure 7_

실제 데이터셋의 label 예시. COLMAP이나 LiDAR로 얻은 depth는 비어 있거나 noise가 많다.

\[문제점\]<br>
실제 이미지의 label이 부정확하다.

\[해결책\]<br>
Teacher 모델 (Depth-Anything-3-Teacher)은 **합성 데이터로만 학습**한다. DA2 teacher와 비교해 달라진 점은 다음과 같다.

- **합성 데이터 확대**: Hypersim, TartanAir, IRS, vKITTI2, BlendedMVS, SPRING, Objaverse, Trellis 등 20개 합성 데이터셋으로 크게 늘려 실내, 실외, 물체 중심, in-the-wild 장면을 덮는다.
- **Disparity → Depth**: DA2는 scale-shift-invariant disparity를 예측했지만, metric depth나 multi-view geometry처럼 depth 공간에서 바로 계산하는 downstream task에는 depth가 더 편하다.
- **Exponential depth**: depth는 disparity보다 가까운 거리를 구분하는 힘이 약해, 선형 depth 대신 지수 (exponential) depth를 예측한다.
- **Loss**: gradient loss와 global-local loss에 거리 가중 surface normal loss를 더하고, GT가 없는 하늘과 물체 배경은 sky mask, object mask를 함께 예측해 처리한다.

**② Teaching Depth Anything 3.**

\[문제점\]<br>
합성 이미지는 pose와 label이 정확하지만 실제 이미지와 domain gap이 있다. 실제 이미지는 camera pose 일반화에 꼭 필요하지만 **label이 부정확하다**.

\[해결책\]<br>
Teacher가 실제 이미지의 relative depth를 만들고, 이를 COLMAP이나 센서로 얻은 noisy metric depth에 **RANSAC으로 scale-shift 정렬해 label로 쓴다**.

$$
(\hat{s}, \hat{t}) = \arg\min_{s>0,\,t}\sum_{p\in\Omega} m_p\left(s\,\tilde{\mathbf{D}}_p + t - \mathbf{D}_p\right)^2, \qquad \mathbf{D}^{T\to M} = \hat{s}\,\tilde{\mathbf{D}} + \hat{t}
$$

$$\tilde{\mathbf{D}}$$는 teacher의 relative depth, $$\mathbf{D}$$는 sparse depth, $$m_p$$는 유효 픽셀 mask이다. 쉽게 말해, teacher depth의 detail은 살리고 크기와 위치 (scale, shift)만 실제 측정값에 맞추는 것이다. 학습 200k step 중 120k step부터 GT depth 대신 이 teacher label로 supervision을 바꾼다.

![Datasets used in Depth Anything 3](table9.webp){: w="600" }
_출처: 원 논문 Table 9_

학습과 평가에 쓴 데이터셋. Pose-geometry 학습에는 합성, LiDAR, COLMAP 등 출처가 다른 21개 데이터셋을 쓴다.

**③ Training objective.**

$$
\mathcal{L} = \mathcal{L}_D(\hat{\mathbf{D}}, \mathbf{D}) + \mathcal{L}_M(\hat{\mathbf{R}}, \mathbf{M}) + \mathcal{L}_P(\hat{\mathbf{D}} \odot \mathbf{d} + \mathbf{t}, \mathbf{P}) + \beta\mathcal{L}_C(\hat{\mathbf{c}}, \mathbf{v}) + \alpha\mathcal{L}_{grad}(\hat{\mathbf{D}}, \mathbf{D})
$$

$$\mathcal{L}_D$$는 confidence를 반영한 depth loss, $$\mathcal{L}_M$$은 ray map loss, $$\mathcal{L}_P$$는 depth와 ray로 만든 3D 점의 loss, $$\mathcal{L}_C$$는 camera head loss, $$\mathcal{L}_{grad}$$는 depth gradient loss이고 $$\alpha = \beta = 1$$이다. Loss 계산 전에 모든 GT를 유효한 point map 크기의 평균으로 나눠 scale을 맞춘다. 쉽게 말해, depth, ray, 그 둘로 만든 3D 점, camera를 모두 GT와 맞추고 경계가 날카롭도록 depth의 변화량까지 맞추는 loss이다.

### 4. Application: Feed-Forward 3D Gaussian Splatting

Consistent한 depth 추정은 downstream 3D vision task를 크게 도울 수 있다는 생각에서, feed-forward novel view synthesis (FF-NVS)를 시연 task로 고른다.

**① Pose-conditioned feed-forward 3DGS.**

- 최소 모델링: DPT head 하나 (GS-DPT)를 추가하고 fine-tuning한다.
- GS-DPT: 픽셀마다 camera space의 3D Gaussian 하나를 예측한다.
  - 속성 (불투명도, 회전, 크기, 색) ← GS-DPT (불투명도만 confidence head가 예측)
  - 위치 $$\mathbf{P}$$ ← 예측한 depth를 월드 좌표로 unproject
- 이 Gaussian들을 rasterize해 주어진 camera pose의 새 시점 이미지를 만든다.
- 새 시점 이미지의 photometric loss (MSE, LPIPS)와 입력 view의 scale-shift-invariant depth loss로 학습한다.

**② Pose-adaptive feed-forward 3DGS.** (arXiv v1 내용, 최종본에서는 빠짐) Pose가 있든 없든 쓸 수 있는 유연한 구조이다.

- 조건: (1) 모든 3DGS parameter를 local camera space에서 예측하고, (2) backbone이 pose가 있는 이미지와 없는 이미지를 모두 처리해야 한다. DA3 backbone은 둘 다 만족한다.
- 방법: pose를 알면 예측한 depth와 3DGS를 주어진 pose에 scale을 맞춰 월드 좌표로 옮기고, 모르면 DA3가 예측한 pose로 옮긴다.
- 학습: 학습이 불안정해지지 않도록 **DA3 backbone을 사전학습 가중치로 초기화한 뒤 고정**하고 GS-DPT head만 학습한다. 고해상도 입력에는 적은 view, 저해상도 입력에는 많은 view를 짝지어 해상도와 view 수를 바꿔 가며 학습한다.
- In-the-wild robustness를 위해 depth offset을 추가로 예측하고, Gaussian 색을 spherical harmonics 계수로 바꾼다.

---

## Results

- 자세한 실험 설정은 논문 참고

### 1. Pose Estimation

![Pose accuracy comparison](table1.webp)
_출처: 원 논문 Table 1_

Auc3/Auc30은 view 쌍의 상대 회전·이동 각도 오차로 만든 정확도 곡선을 threshold 3°/30°까지 적분한 값이다 (높을수록 좋음).

- DA3-Giant가 거의 모든 지표에서 가장 좋다. Feed-forward 모델들과 비교해 **Auc3가 상대적으로 최소 8% 높고**, ScanNet++에서는 2등보다 33% 높다.
- COLMAP, GLOMAP 같은 고전 SfM은 촘촘하고 texture가 많은 DTU에서는 강하지만 (GLOMAP Auc3 96.8), view가 sparse하거나 texture가 없는 장면에서는 크게 떨어진다 (HiRoom Auc3: COLMAP 13.0 vs DA3 81.7).

### 2. Geometry Estimation

![Reconstruction accuracy comparison](table2.webp)
_출처: 원 논문 Table 2_

예측한 pose와 depth로 point cloud를 만들어 GT와 비교한다. 지표는 F1 (DTU만 Chamfer Distance, mm)이고, w/o p.는 pose 없이, w/ p.는 GT pose를 준 경우이다.

- Pose가 없는 5개 설정 모두에서 feed-forward 모델 중 가장 좋고, 평균적으로 **VGGT보다 23.6%, Pi3보다 16.7% 높다**.
- DA3-Large (0.30B)는 VGGT (0.90B)보다 **3배 작지만 10개 설정 중 5개에서 VGGT를 이긴다**.
- Pose를 주면 7Scenes를 빼고 대부분 좋아진다. 다만 pose 조건에서는 모델을 키워 얻는 이득이 작아, **pose 추정이 depth 추정보다 큰 모델을 더 필요로 한다**는 것을 보여 준다.
- 고전 방법은 DTU처럼 아주 촘촘한 데이터에서는 경쟁력이 있지만 (GLOMAP+PM 1.62mm), 조금만 sparse해져도 무너진다 (ETH3D F1: COLMAP+PM 20.7 vs DA3 74.4).

### 3. Monocular Depth Estimation

![Monocular depth comparison](table3.webp){: w="400" }
_출처: 원 논문 Table 3_

- 표준 단안 depth benchmark에서도 **VGGT와 DA2를 앞선다** (평균 순위 2.20).
- 참고로 함께 넣은 teacher 모델은 모든 데이터셋에서 가장 좋다 (평균 순위 1.00).

### 4. Novel View Synthesis

![NVS comparison](table4.webp){: w="460" }
_출처: 원 논문 Table 4_

12장의 context view로 새 시점을 렌더링한다. 모든 모델을 DL3DV-10K로 같은 조건에서 학습했다.

- Geometry 모델 기반 framework (Fast3R, MV-DUSt3R, VGGT, DA3)가 전용 feed-forward 3DGS 모델 (pixelSplat, MVSplat, DepthSplat)보다 일관되게 좋다. 단순한 backbone + DPT head가 **복잡한 전용 설계를 이긴다**.
- 그 안에서는 geometry 추정 능력과 NVS 성능이 같이 움직이며, **DA3가 모든 데이터셋에서 가장 좋다**.
- 모든 모델이 DL3DV에서 다른 데이터셋보다 훨씬 좋다. 3DGS 기반 NVS가 장면 내용보다 camera 궤적과 pose 분포에 민감하다는 뜻이다.

### 5. Sufficiency of the Depth-Ray Representation

![Ablation of prediction targets](table5.webp)
_출처: 원 논문 Table 5_

ViT-L, camera condition token 없이 예측 대상 조합만 바꿔 비교했다.

- Point cloud만 예측하면 (pcd) camera pose 추정에 필요한 기하 구조가 부족해 성능이 낮다 (평균 Auc3 31.6). Depth + ray에 pcd를 더해도 (depth + ray + pcd) 일부 데이터셋만 좋아지고 나머지는 나빠진다.
- Depth + ray만으로 균형 잡힌 성능 (평균 Auc3 36.0, F1 56.4)을 내며, **depth + cam보다 Auc3가 155% 이상 높다**.
- 최종 모델은 depth + ray + cam을 쓴다. Camera head는 계산 비용이 backbone의 약 0.1%이고, ray map에서 pose를 푸는 것보다 훨씬 빠르기 때문이다.

### 6. Sufficiency of a Single Plain Transformer

![Ablation of single transformer](table6.webp)
_출처: 원 논문 Table 6_

비슷한 parameter 수로 세 구조를 비교했다. (a) 제안 구조 (ViT-L), (b) VGGT식 구조 (ViT-B 두 개를 쌓아 block 수 3배), (c) Full Alt. (모든 layer에서 cross-view와 within-view를 번갈아 함).

- VGGT식 구조는 **모든 지표에서 크게 떨어진다** (HiRoom Auc3 39.2 → 3.72). 제안 구조는 backbone 전체가 사전학습된 반면, VGGT식은 block의 2/3가 사전학습되지 않았기 때문으로 본다.
- Full Alt.도 7Scenes F1을 빼고 거의 모든 지표에서 나빠, 앞쪽 layer는 within-view만 쓰는 **부분 교차가 더 효과적**이다.

---

## Conclusion

본 논문은 평범한 transformer 하나를 depth와 ray target, teacher-student supervision으로 학습하면 **복잡한 구조 없이 any-view geometry를 통합할 수 있음**을 보였다.

(1) Scale-aware depth, 픽셀별 ray, input-adaptive cross-view attention으로 사전학습 feature를 그대로 물려받으면서도 가볍고 확장하기 쉽다.

(2) 새 visual geometry benchmark에서 pose와 reconstruction 모두 SOTA를 달성했고, 큰 모델과 작은 모델 모두 기존 모델을 앞선다.

(3) 같은 backbone으로 feed-forward NVS까지 수행한다.

**Future work.** Dynamic scene으로 확장, 언어와 상호작용 신호 결합, 더 큰 규모의 사전학습으로 geometry 이해와 world model을 잇는 것이 다음 과제이다.
