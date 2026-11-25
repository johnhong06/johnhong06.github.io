---
title: "Honey, I Shrunk the Arc de Triomphe! (ECCV 2026)"
date: 2026-11-25 11:00:00 +0900
categories: [논문리뷰]
tags: [monocular depth estimation, metric depth, dataset, 3d reconstruction]
math: true
description: "최신 metric depth 모델은 멀리 있는 큰 구조물을 실제보다 훨씬 작게 예측한다. 원인은 모델 구조가 아니라 학습 데이터! 인터넷 사진과 stereo 영상에서 실제 크기를 복원한 데이터셋 MetricScenes로 fine-tuning하면 해결된다."
media_subpath: /assets/img/posts/metricscenes/
---

**ECCV 2026.** [Paper](https://arxiv.org/abs/2606.02379) [Project Page](https://metricscenes.github.io/) [Github](https://github.com/MetricScenes/MetricScenes)<br>
Yuanbo Xiangli, Hanyu Chen, Xueqing Tsang, Noah Snavely<br>
Cornell University \| Shanghai Jiao Tong University<br>
26 Jun 2026 (arXiv v2)

핵심 한 줄: **최신 metric depth 모델은 멀리 있는 큰 구조물을 실제보다 훨씬 작게 예측한다. 원인은 모델 구조가 아니라 학습 데이터!** 인터넷 사진과 stereo 영상에서 실제 크기를 복원한 데이터셋 MetricScenes로 fine-tuning하면 해결된다.

![Scale-collapse on the Arc de Triomphe](fig1.webp)
_출처: 원 논문 Figure 1_

MoGe-2 같은 SOTA monocular geometry estimation (MGE) 모델도 관광 사진 속 개선문의 폭을 18.8m로 예측한다 (실제 44.8m). 저자들은 원인을 metric-scale 학습 데이터가 주행 LiDAR, 실내 스캔, synthetic에 치우친 데서 찾는다. Internet photo collection (AerialMegaDepth, MegaScenes)과 stereo 영상 (Stereo4D)에서 geo-tag와 stereo baseline으로 absolute scale을 복원해 MetricScenes를 만들고, depth map의 빈 곳은 two-stage Poisson completion으로 채운다. 이 데이터로 MoGe-2를 fine-tuning한 WildMoGe는 **개선문을 43.4m로 예측하면서 기존 benchmark 성능도 유지**한다.

---

## Introduction

한 장의 이미지에서 장면의 absolute scale을 정하는 것은 본질적으로 ill-posed 문제이다. 개선문 관광 사진만 보고는 실제 랜드마크인지 정교하게 만든 미니어처인지 완전히 확신할 수 없다. 그래도 사람은 크기를 아는 기준 물체 (예: 사람) 같은 시각적 단서로 scale을 짐작한다. 최신 MGE 모델도 semantic prior로 같은 추론을 할 수 있어야 한다.

하지만 저자들은 SOTA 모델이 in-the-wild 장면에서 scale을 자주 틀린다는 것을 발견했고, 이를 **scale-collapse**라 부른다. Fig. 1처럼 사람이라는 분명한 기준이 있는데도, MoGe-2는 가까운 관광객의 키는 그럴듯하게 예측하면서 멀리 있는 개선문은 폭 18.8m로 예측한다.

본 논문의 목표는 zero-shot 사진에서도 먼 구조물의 metric scale을 맞히는 것이며, 이를 **모델 구조가 아닌 학습 데이터 관점**에서 접근한다. 문제점과 해결책은 다음과 같다.

\[문제점1\]<br>
실제 환경의 metric-scale 데이터셋은 장비 제약 때문에 두 종류에 치우쳐 있다.

(1) KITTI, Argoverse 2 같은 **LiDAR 데이터셋은 차량에 달린 센서로 찍은 거리 장면뿐이라 다양성이 부족**하다.

(2) ScanNet++, ARKitScenes 같은 **active 3D scanner 데이터는 유효 거리가 짧아 실내 같은 작은 공간에 한정**된다. **Synthetic 데이터셋은 다양한 시점과 움직임을 주지만, 절차적 규칙과 한정된 3D asset에 의존해 실제 세계의 metric 다양성을 담지 못한다.**

\[해결책1\]<br>
흔히 구할 수 있는 Internet photo collection과 stereo 영상에서 metric depth를 뽑아 MetricScenes를 만든다. 기존 데이터셋에 없던 환경과 semantic의 다양성을 가지며, **geo-tag된 이미지와 stereo camera baseline으로 실제 물리적 크기를 복원**한다.

\[문제점2\]<br>
SfM과 MVS는 여러 장의 사진에 공통으로 보이는 것만 3D로 복원한다. 그래서 건물은 복원되지만, 사진마다 바뀌는 사람 같은 전경은 **depth map에 구멍으로 남는다**.

이 구멍은 MoGe-2 같은 monocular 모델의 예측을 빌려 채운다 (Poisson completion). 그런데 MoGe-2의 예측은 가까운 사람은 맞고 먼 건물은 작게 나온 상태이다. 이 예측을 건물의 실제 크기에 맞춰 늘리면 **사람까지 같이 커져서**, 키가 3m를 넘는 사람이 나오거나 사람과 건물의 경계가 뭉개진다.

\[해결책2\]<br>
Two-stage edge-aware Poisson completion을 제안한다. **배경의 metric 정렬과 전경 합성을 분리**해, 경계는 선명하게 유지하고 scale은 장면 전체에서 일관되게 맞춘다.

Contribution은 다음과 같다.

- SOTA metric MGE 모델이 in-the-wild 장면에서 먼 구조물을 작게 예측하는 scale-collapse 현상을 짚고, 원인을 학습 데이터에서 찾는다.
- Internet photo collection과 stereo 영상에서 만든 real-world metric-scale 데이터셋 MetricScenes를 제안한다.
- 전경과 배경의 scale 충돌을 피하는 two-stage edge-aware Poisson completion을 제안한다.
- MoGe-2를 fine-tuning한 WildMoGe가 in-the-wild 장면에서 scale-collapse를 크게 줄이면서 기존 benchmark 성능을 유지함을 보인다.

---

## Generating Metric-Scale 3D Data

목표는 먼 곳까지 실제 크기 (m)가 맞는 depth 정답을 만드는 것이다. 출처는 Internet photo collection (AerialMegaDepth, MegaScenes)과 stereo 영상 (Stereo4D) 두 가지이고, 풀어야 할 문제도 두 가지이다.

- 실제 크기: 사진으로 복원한 3D는 모양은 맞지만 축척을 모른다. **확실히 아는 길이 하나** (인터넷 사진은 GPS 위치, stereo 영상은 두 렌즈 사이 거리)로 장면 전체의 배율을 정한다.
- 빈 곳: 복원한 depth map에는 구멍이 많다. **크기는 정답 depth에서, 모양은 MoGe-2 예측에서** 가져와 채운다.

학습 정답으로 쓰는 데이터라, 단계마다 틀린 부분을 지우거나 걸러낸다.

### 1. Internet Photo Collections

AerialMegaDepth와 MegaScenes의 관광 사진, 역사 자료, 전문 사진을 쓴다. 3D 복원 → 틀린 부분 지우기 → 실제 크기 맞추기 순서로 처리한다.

![MVS depth and filtering on Internet photos](fig2.webp)
_출처: 원 논문 Figure 2_

**① 3D 복원 (SfM → MVS).** 둘 다 같은 장소를 찍은 사진 여러 장으로 3D를 복원하는 기술이고, 순서대로 이어서 쓴다.

- SfM (Structure from Motion): 사진들 사이에서 같은 점을 찾아 짝짓고, 각 사진의 카메라 위치와 듬성듬성한 3D 점들을 구한다. 모양과 비율만 맞고 **실제 크기는 모른다**.
- MVS (Multi-View Stereo): SfM이 구한 카메라 위치를 써서, 옆 사진과 색이 가장 잘 맞는 depth를 픽셀마다 골라 dense depth map을 만든다.
- AerialMegaDepth는 데이터셋이 제공하는 SfM 결과를 그대로 쓰고, MegaScenes는 MASt3R-SfM으로 다시 복원한다.

**② 틀린 부분 지우기.** 이 depth map이 그대로 학습 정답이 되므로, 틀린 부분은 남기지 않는다.

- Doppelganger: 건물은 앞면과 뒷면처럼 닮은 면이 많아, 서로 다른 면이 같은 면으로 잘못 짝지어지면 3D 모델이 겹쳐서 망가진다. MegaScenes를 다시 복원할 때 두 사진이 정말 같은 면을 찍었는지 판별하는 Doppelganger classifier로 잘못된 쌍을 걸러낸다.
- 움직이는 물체: MVS는 여러 사진에서 장면이 같다고 가정하므로, 사람처럼 그 사진에만 있는 물체는 depth가 불안정하다. Depth 분산이 큰 pixel을 지운다 (Fig. 2 노란 상자).
- Depth bleeding: 가느다란 앞쪽 물체에 배경의 depth가 번져 들어간 영역은 MoGe-2 예측과 비교해 지운다 (빨간 상자).

지운 자리는 3절의 depth completion으로 채운다.

**③ 실제 크기 맞추기 (geolocation).**

- AerialMegaDepth: Google Earth에서 위치를 지정해 렌더링한 view (이미 m 단위)와 인터넷 사진을 함께 복원한 데이터라, 처음부터 실제 크기가 맞다.
- MegaScenes: 크기 정보가 없어, 온라인 지도 서비스에서 **GPS 위치를 아는 (geo-tag된) street view 사진**을 가져와 3D 모델에 끼워 넣는다.

MegaScenes는 다음 순서로 크기를 맞춘다.

1. 사진 모으기: 랜드마크 주변 도로를 따라 약 100개 위치에서, 장면을 향하고 가려지지 않은 street view 사진을 모은다. 위치가 더 정확한 전문 제공자의 사진을 우선한다.
2. 3D 모델에 등록: MASt3R로 street view 사진을 기존 SfM 모델과 매칭하고, Doppelganger classifier로 잘못된 대응을 지운다. COLMAP model aligner로 장면의 수직, 수평 구조를 축에 맞춰 둔다 (Manhattan World frame).
3. 배율 구하기: 모델 속 카메라 위치와 실제 GPS 위치 (ECEF 좌표) 사이의 similarity transformation (배율, 회전, 이동)을 RANSAC으로 구한다. 이 배율이 곧 **"모델 속 1 = 실제 몇 m"**이다.
4. 검사: 배율이 비현실적이거나, 위치 오차가 크거나, RANSAC inlier 비율이 낮은 scene은 버린다.

쉽게 말해, 크기를 모르는 모형 위에 위치를 아는 깃발 몇 개를 꽂고, 깃발 사이의 실제 거리로 모형 전체의 축척을 정하는 것이다.

### 2. Stereo Imagery

Stereo4D는 VR180 장비로 찍은 real-world stereo 영상 데이터셋이다. 여기서는 **두 렌즈 사이의 실제 거리 (baseline)가 크기의 기준**이다. 원래 Stereo4D는 대부분의 VR180 카메라 baseline을 약 6.3cm (사람의 평균 동공 간 거리)로 가정하지만, 실제로는 카메라마다 다르다. 그래서 영상 설명에 카메라 구성이 적혀 있어 baseline을 정확히 아는 영상만 쓴다. 다시 복원 → 실제 크기 맞추기 → 거르기 순서로 처리한다.

![Stereo4D reconstruction comparison](fig3.webp)
_출처: 원 논문 Figure 3_

**① 다시 복원 (π³).** Stereo4D가 원래 제공하는 geometry (optical flow, SEA-RAFT)는 stereo calibration 오차 때문에 왜곡된다. 평행한 건물 벽이 먼 한 점으로 모이는 식이다 (Fig. 3 위, magenta 상자). Stereo matching (FoundationStereo)도 마찬가지라, camera calibration, pose, depth를 multi-view reconstruction으로 다시 구한다.

- Clip마다 가운데 target frame과 주변 $$N=16$$ frame (temporal stride $$K=3$$)을 뽑아, frame별로 따로 풀지 않고 $$N+1$$개 stereo pair 전체를 한꺼번에 넣는다.
- π³, Depth Anything 3, MapAnything을 비교해 geometry가 가장 안정적이고 local detail이 선명한 π³를 골랐다 (Fig. 3 위, cyan 상자).

**② 실제 크기 맞추기 (baseline).** π³의 결과는 크기를 모르는 임의의 좌표계에 있다. 복원된 두 렌즈 사이의 평균 거리가 실제 baseline $$b_{\text{gt}}$$가 되도록 배율 $$s$$를 구해 depth 전체에 곱한다.

$$
s = \frac{b_{\text{gt}}}{\frac{1}{N+1}\sum_{i=0}^{N} \lVert \mathbf{t}_{\text{L}}^{(i)} - \mathbf{t}_{\text{R}}^{(i)} \rVert_2}, \qquad d_{\text{metric}} = s \cdot d_{\text{pred}}
$$

$$\mathbf{t}_{\text{L}}^{(i)}$$와 $$\mathbf{t}_{\text{R}}^{(i)}$$는 $$i$$번째 frame에서 추정한 왼쪽, 오른쪽 카메라의 translation이다.

쉽게 말해, 두 렌즈 사이의 실제 거리를 알고 있으니, 모델이 예측한 두 렌즈 사이 거리가 그 값이 되도록 depth 전체에 같은 배율을 곱한다.

**③ 거르기.** Clip 안의 frame은 서로 비슷해서 clip당 한 frame (center-left frame)만 쓰고, 다음 경우는 버린다.

- 화질: 어둡거나 (median intensity) motion blur가 있는 (Tenengrad gradient energy) frame
- 좌우 일관성: 좌우 depth reprojection check에서 valid pixel의 10% 넘게 맞지 않는 frame
- Calibration: 배율을 맞춘 뒤에도 baseline이 실제와 10mm 넘게 다르거나, 두 카메라의 상대 회전이 1°를 넘는 frame
- MoGe-2와 비교: MoGe-2 예측 대비 scale이 2배를 넘거나 0.5배보다 작은 frame. 가까운 장면 (보행자, 건물)은 MoGe-2가 잘 맞히므로, 이만큼 다르면 복원이 잘못된 것으로 본다.

### 3. Depth Fusion and Completion

앞 단계의 depth map은 두 출처가 서로 반대쪽이 비어 있다.

- Internet photo: 사람 같은 전경과 transient 물체가 비어 있다 (Fig. 2).
- Stereo 영상: confidence가 낮은 먼 배경이 비어 있다 (Fig. 3).

MoGe-2 같은 monocular 모델은 반대로 빈틈없는 depth를 내지만 먼 곳의 크기를 작게 잡는다. 그래서 크기는 정답 depth에서, 모양은 MoGe-2에서 가져온다. 정답 depth는 scale을 잡는 고정점 (anchor), MoGe-2 예측은 빈 곳을 채우는 guidance로 쓴다.

**Single-pass completion.** MoGe-2가 제안한 log 공간 Poisson completion이다. 정답 depth를 anchor로 고정하고, 빈 영역은 MoGe-2 예측의 굴곡 (gradient)을 따라 채운다.

$$
\min \sum_{i \in \Omega} \lVert \nabla(\log d^*_i) - \nabla(\log \hat{d}_i) \rVert^2, \quad \text{s.t.} \quad d^*_i = d_i,\ \forall i \in \partial\Omega
$$

$$d^*_i$$는 구하려는 depth, $$\hat{d}_i$$는 gradient guidance를 주는 예측 depth (여기서는 MoGe-2), $$d_i$$는 빈 영역의 경계에 고정된 GT anchor이다.

Stereo4D는 이 방식으로 잘 채워진다. 하지만 인터넷 사진 (AerialMegaDepth, MegaScenes)에서는 MoGe-2의 scale-collapse 때문에 **한 번에 풀면 실패**한다 (Fig. 4).

![Single-pass Poisson completion failures](fig4.webp)
_출처: 원 논문 Figure 4_

- **BG only** (배경만 anchor): MoGe-2는 전경은 맞고 먼 건물만 작게 예측한 상태이다. Solver는 depth map을 하나로 이어진 면으로 다루므로, 배경을 실제 크기로 늘린 효과가 전경까지 퍼져 사람도 같이 커진다. 키가 3.26m인 사람이 나온다.
- **FG + BG** (배경은 MVS, 전경은 MoGe-2를 함께 anchor): 서로 다른 두 scale을 한 번에 맞추려다 보니, anchor가 없는 사이 영역 (예: 건물 아랫부분)이 전경 쪽으로 휘고 depth bleeding이 생긴다 (초록 상자).

**Two-stage edge-aware completion.** 저자들은 실패 원인을 anchor가 듬성듬성한 (sparse) 데서 찾는다. 기준이 듬성듬성하면, 배경을 앞으로 당기는 MoGe-2의 gradient와 metric anchor를 함께 만족시키지 못해 어딘가가 휜다. 그래서 배경을 먼저 빽빽하게 채워 기준을 만든 뒤 전경을 붙인다.

![Two-stage edge-aware completion](fig5.webp)
_출처: 원 논문 Figure 5_

- **Stage 1: Background completion.** 배경의 depth만 anchor로 위 식을 푼다 (BG only와 같다). 배경은 일관된 metric scale로 빽빽하게 채워진다. Scale이 틀린 전경 자리의 결과는 버리고, 채워진 배경만 다음 단계로 넘긴다.
- **Stage 2: Edge-aware composition.** Stage 1의 배경과 MoGe-2의 전경을 합쳐 anchor로 쓰고, 경계에서는 서로 영향을 주지 않도록 edge weight를 넣어 한 번 더 푼다.

$$
\min \sum_{i \in \Omega} \sum_{j \in \mathcal{N}(i)} w_{ij} \lVert (\log d^*_i - \log d^*_j) - (\log \hat{d}_i - \log \hat{d}_j) \rVert^2
$$

$$w_{ij}$$는 depth가 급격히 끊기는 경계에서 0에 가까워진다. 배경은 Stage 1 결과로 dense하게 고정되어 scale이 유지되고, edge weight 덕분에 전경은 선명한 경계로 합쳐진다. Fig. 4에서 **사람의 키가 3.26m에서 1.81m로** 돌아온다.

쉽게 말해, 배경은 MVS (실제 크기)를 믿고 전경은 MoGe-2 (가까운 것은 잘 맞힘)를 믿는다. 한 번에 풀면 두 scale이 충돌하니, **먼저 배경만 채운 뒤 그 위에 전경을 경계를 끊어서 붙인다**.

---

## Results

- 자세한 실험 설정은 논문 참고

### 1. MetricScenes와 WildMoGe

- AerialMegaDepth: 134 scene, 47,579장 (real-world 이미지만)
- MegaScenes: 356 scene, 29,583장
- Stereo4D: 1,725 video, 22,549 frame (clip당 1 frame)
- Validation과 test용으로 AerialMegaDepth 10 scene, MegaScenes 10 scene, Stereo4D 10 video를 따로 뺀다.

Ground view와 aerial view, 도시와 자연, 실내와 실외를 고루 포함한다.

WildMoGe는 MoGe-2 ViT-Large-Normal을 MetricScenes로 10,000 iteration (batch size 32, 약 3 epoch) fine-tuning한 모델이다. Learning rate는 backbone $$1\times10^{-6}$$, 나머지 $$1\times10^{-5}$$로, 사전학습된 geometric prior에서 크게 벗어나지 않게 한다.

### 2. In-the-wild 장면

학습에 없는 랜드마크 장면에서 폭을 잰다. 첫 열의 실측값은 Google Maps 측정 도구로 얻은 값이다.

![In-the-wild landmark width comparison](fig6.webp)
_출처: 원 논문 Figure 6_

**WildMoGe는 네 장면 모두 실측에 가깝고, MoGe-2, Depth Anything 3, Metric3D v2는 먼 구조물을 대체로 작게 예측한다.**

- 위에서부터 네 장면의 실측이 약 32.4m / 18.2m / 38.8m / 46.5m일 때 WildMoGe는 31.4m / 17.9m / 37.8m / 46.7m, MoGe-2는 22.8m / 13.3m / 23.8m / 24.6m이다. UniDepth v2는 49.8m / 15.3m / 34.5m / 30.0m로 더 현실적이지만 차이가 있고, Depth Pro는 3~8m로 자릿수가 다르다.

### 3. 일반 장면

기존 학습 데이터에 흔한 실내, 거리 장면에서 MoGe-2와 비교한다.

![Common scene scale comparison](fig7.webp)
_출처: 원 논문 Figure 7_

**흔한 장면에서는 MoGe-2와 scale이 거의 같고, WildMoGe가 무조건 크게 예측하는 것도 아니다.**

- 문 높이는 둘 다 2.1m, 차 길이는 MoGe-2 3.3m, WildMoGe 3.2m이다. ETH3D courtyard의 책상 다리 (실측 72cm)는 MoGe-2 81cm, WildMoGe 71.6cm로, WildMoGe가 더 작으면서 더 정확하다.

### 4. 정량 평가

Standard benchmark는 MoGe-2를 따라 10개 데이터셋 (metric geometry는 그중 metric annotation이 있는 7개)에서 평가하고, MetricScenes test set은 Poisson completion 전의 partial dense depth를 GT로 쓴다. 지표는 상대 오차 Rel과 inlier 비율 δ1이며, 위첨자 p는 point map, d는 depth를 뜻한다.

![Quantitative comparison with MoGe-2](table1.webp)
_출처: 원 논문 Table 1_

**MetricScenes test set에서는 모든 지표에서 MoGe-2를 앞선다. Standard benchmark에서는 relative geometry는 비슷하고 metric geometry는 조금 낮다.**

- Metric depth (w/o GT Cam)를 MoGe-2 → WildMoGe로 보면, MetricScenes test set에서 Rel 37.2 → 26.5, δ1 37.7 → 73.8이고, standard benchmark에서는 Rel 15.7 → 15.6, δ1 76.8 → 73.4이다.
- 저자들은 standard benchmark의 하락이 domain shift 때문이며 joint training으로 줄일 수 있다고 본다. 또 MetricScenes test set의 GT는 같은 pipeline으로 만든 label이라, 이 결과가 덜 확정적일 수 있다고 밝힌다.

### 5. Ablation: 학습 subset

MetricScenes의 subset별로 MoGe-2를 fine-tuning한다 (모두 3 epoch).

![Training subset ablation](table2.webp)
_출처: 원 논문 Table 2_

**출처마다 기여가 다르고, 세 출처를 모두 쓴 All이 MetricScenes test set의 모든 지표에서 가장 좋다.**

- MetricScenes test set의 metric depth δ1 (w/o GT Cam)은 Stereo4D only 39.3, AerialMD. only 58.7, MegaSc. only 53.0, AerialMD.+MegaSc. 65.0, All 73.8이다. Standard benchmark에서는 Stereo4D가 boundary sharpness (F1 15.6), AerialMegaDepth가 metric depth δ1 (w/o GT Cam 78.9)에서 가장 높다.

---

## Conclusion

본 논문은 in-the-wild 장면의 monocular metric depth/geometry estimation에서 scale-collapse 문제를 짚었다. 최신 모델은 standard benchmark에서는 뛰어나지만, 먼 랜드마크나 넓은 풍경에서는 실제 크기를 유지하지 못한다. 저자들은 이를 **장비 제약으로 scale 다양성이 부족한 학습 데이터의 문제**로 본다.

(1) Internet photo collection과 stereo 영상에서 geo-tag와 camera baseline으로 metric scale을 복원해 MetricScenes를 만들자.

(2) 전경과 배경의 scale이 충돌하지 않게 two-stage Poisson completion으로 depth를 채우자.

이 데이터로 MoGe-2를 fine-tuning하면 standard benchmark 성능을 유지하면서 in-the-wild 장면의 scale-collapse를 줄일 수 있다.
