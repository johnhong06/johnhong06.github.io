---
title: "Depth Anything V2 — 근경 물체가 원거리 추정을 지배한다"
date: 2026-09-09 15:16:00 +0900
categories: [실험]
tags: [monocular depth estimation, depth anything v2, real-time inference]
description: "① 근경 물체 하나가 그 뒤 전부를 누른다 (영상 1·2, Large 고정). ② 폰 카메라 실시간 S/B/L (영상 3). V2 추론은 모두 fp32."
permalink: /depth-anythingv2-test1/
---

## 이 페이지의 목적 {#purpose}

| | 다루는 것 | 고정 | 변수 |
|---|---|---|---|
| **1부** (영상 1·2) | 근경 물체가 원거리를 누른다 | Large · fp32 | **입력 화면 구성** |
| **2부** (영상 3) | 실시간 처리 성능 | 장면 · fp32 · 15초 | **모델 크기 S/B/L** |

1부는 모델 크기 비교가 아니다. 둘 다 **V2-Large · fp32** 하나로만 돌렸다.

| 패널 | 내용 |
|---|---|
| 왼쪽 | 입력 |
| 가운데 | **전체 프레임** 추론 |
| 오른쪽 | **근경을 잘라낸 프레임** 추론 (잘린 곳은 검게) |

어디를 자를지는 **최근접이 어디냐**로 정한다. 크롭 띠는 원래 있던 세로 위치에 놓아 가운데 패널과 바로 대조된다.

## 관찰된 특성 {#finding}

V2 출력은 **상대 역깊이**(`1/z`)라 **가장 가까운 물체가 값 범위를 독점**한다.
대상의 값 위치는 절대 거리가 아니라 **최근접과의 비율**로 정해진다.

| 최근접 | 대상 | 비율 | 대상의 값 위치 |
|---|---|---:|---:|
| 0.3 m | 30 m 드론 | 100 : 1 | **1.0 %** |
| 3 m | 30 m 드론 | 10 : 1 | **10 %** |
| 2 m (논문 데모) | 100 m 건물 | 50 : 1 | 2.0 % |

**공식 데모의 배경이 선명한 것은 배경이 가까워서가 아니라 렌즈 코앞에 물체가 없기 때문이다.**

### 실측 {#measure}

영상 1에서 드론이 검출된 **256프레임 전량**, 크롭 조건만 바꿔 재추론.
`contrast` = (드론 − 주변 하늘) 깊이 차 ÷ 전체 값 범위, 256프레임 중앙값.

| 조건 | contrast | 전체 대비 |
|---|---:|---:|
| 전체 프레임 | 0.19 % | 1× |
| 위 25 % 제거 (천막만) | 0.48 % | 2.5× |
| 아래 50 % 제거 (장비·잔디만) | 2.63 % | 13.6× |
| **위 25 % + 아래 50 % 제거** | **6.77 %** | **35×** |

> **0.19 % → 6.77 %, 약 35배.** 모델·가중치는 그대로, 바뀐 것은 화면 구성뿐이다.
{: .prompt-tip }

## 영상 1 — 근경이 위(천막)·아래(장비) 양쪽 {#v1}

1218×1078 · 1172프레임 · 30 fps · 25 %~50 % 구간만 유지

<p>
  <video class="embed-video file" style="aspect-ratio:auto" src="/depth-anythingv2-test1/assets/video/IMG_E5065.mp4" autoplay muted loop playsinline preload="metadata"></video>
  <em>입력 / 전체 프레임 추론 / 근경 제거 후 추론 — 모두 V2-Large fp32</em>
</p>

### 같은 장면, DAv3 {#dav3}

2520×1120 · 157프레임 · 10 fps · 크롭 없음 · **Depth Anything 3 결과**

<p>
  <video class="embed-video file" style="aspect-ratio:auto" src="/depth-anythingv2-test1/assets/video/drone04_dav3.mp4" autoplay muted loop playsinline preload="metadata"></video>
  <em>왼쪽 입력 / 오른쪽 깊이 — 영상 1과 같은 촬영본, 단 <strong>V2가 아니라 DAv3</strong></em>
</p>

영상 1의 **가운데 패널(V2·전체)** 과 견주면 된다. DAv3도 근경이 값 범위를 크게 가져가지만
크롭 없이도 **건물 윤곽 · 돔 지붕선 · 조명탑 · 산 능선**이 하늘에서 떨어져 나온다.

## 영상 2 — 근경 물체 없이 바닥이 최근접 {#v2}

1080×1920 · 510프레임 · 30 fps · 위 45 % 유지

<p>
  <video class="embed-video file" style="aspect-ratio:auto" src="/depth-anythingv2-test1/assets/video/JPKYE9578.mp4" autoplay muted loop playsinline preload="metadata"></video>
  <em>입력 / 전체 프레임 추론 / 근경 제거 후 추론 — 모두 V2-Large fp32</em>
</p>

## 영상 3 — 실내 실시간 스트리밍, S/B/L {#v3}

640×480 MJPEG · 실측 수신 30.4 fps · 크롭 없음 · fp32 · **각 15초**

<p>
  <video class="embed-video file" style="aspect-ratio:auto" src="/depth-anythingv2-test1/assets/video/indoor/small.mp4" autoplay muted loop playsinline preload="metadata"></video>
  <em><strong>V2-small</strong> — 429프레임 · 28.5 FPS</em>
</p>

<p>
  <video class="embed-video file" style="aspect-ratio:auto" src="/depth-anythingv2-test1/assets/video/indoor/base.mp4" autoplay muted loop playsinline preload="metadata"></video>
  <em><strong>V2-base</strong> — 341프레임 · 22.7 FPS</em>
</p>

<p>
  <video class="embed-video file" style="aspect-ratio:auto" src="/depth-anythingv2-test1/assets/video/indoor/large.mp4" autoplay muted loop playsinline preload="metadata"></video>
  <em><strong>V2-large</strong> — 155프레임 · 10.3 FPS</em>
</p>

세 영상의 재생 시간은 모두 15초로 같고 프레임 수만 다르다 — **large가 끊겨 보이는 것이 곧 성능 차이**다.

| 모델 | 처리 프레임 | 추론 p50 | 추론 p95 | 모델 단독 상한 | 실제 처리 | peak VRAM |
|---|---:|---:|---:|---:|---:|---:|
| small | 429 | 17.0 ms | 40.3 ms | 58.8 FPS | **28.5 FPS** | 354 MB |
| base | 341 | 29.1 ms | 54.5 ms | 34.3 FPS | **22.7 FPS** | 768 MB |
| large | 155 | 96.6 ms | 107.2 ms | 10.3 FPS | **10.3 FPS** | 1,932 MB |

- **small**: 모델 상한 58.8 FPS인데 카메라가 30 fps라 거기서 막힌다 — **스트림이 한계**.
- **large**: 15초 동안 **161프레임을 건너뛴다.** fp32로는 실시간이 안 된다.

> **주의** — 스크립트는 세 경우 모두 병목을 "추론"으로 출력하지만 small은 실제로 스트림 병목이다. 선입력 스레드 때문에 수신 시간이 0.1 ms로 찍혀 대기가 숨겨진다.
{: .prompt-warning }

> 실시간이 필요하면 **fp16**이다. 같은 640×480에서 large fp16은 26 FPS, 깊이맵 차이는 fp32 대비 MAE 0.014 %.
{: .prompt-tip }

## V2-Large · fp32 성능 {#perf}

| 입력 | 모델 입력 텐서 | 추론 p50 | FPS | 30 fps 대비 | peak VRAM |
|---|---|---:|---:|---:|---:|
| 1920×1080 파일 | 518×924 (479k px) | 127.32 ms | 7.9 | 0.27× | 2,145 MB |
| 1080×1920 파일 | 924×518 (479k px) | 126.21 ms | 7.9 | 0.26× | 2,145 MB |
| 640×480 폰 스트림 | 518×686 (355k px) | 96.80 ms | 10.3 | 0.34× | 1,932 MB |
| 1080×864 (영상 2 크롭) | 518×644 (334k px) | 86.38 ms | 11.6 | 0.39× | — |
| 1218×1078 (영상 1 원본) | 518×588 (305k px) | 92.17 ms | 10.8 | 0.36× | 1,843 MB |
| 1080×1056 파일 | 518×532 (276k px) | 70.07 ms | 14.3 | 0.48× | — |
| **1218×270 (영상 1 크롭)** | **112×518 (58k px)** | **37.47 ms** | **26.7** | **0.89×** | — |

- **추론 시간은 소스 해상도가 아니라 모델 입력 텐서를 따른다.** 전처리기가 518 기준으로 맞추므로 1920×1080과 1080×1920이 같은 시간이다.
- **크롭은 품질과 속도를 동시에** 얻는다. 영상 1은 같은 프레임에서 92 ms → 37 ms (305k → 58k px).
- **fp32 Large는 이 GPU에서 30 fps가 안 된다** (최선인 영상 1 크롭도 0.89×). 실시간이면 fp16 또는 입력 축소.
- 크롭 두 행은 `compare_crop.py`, 나머지는 `video_depth.py` / `live_depth.py` 측정값이라 절대 비교는 주의 (같은 프레임을 92.17 ms vs 90.18 ms로 잰다).

## 재현 방법 {#repro}

```bash
# 영상 1 — 위(천막)·아래(장비·잔디) 둘 다 자른다
python src/compare_crop.py --video outputs/test1/IMG_E5065.MOV \
    --model large --dtype fp32 --cut-top 0.25 --keep-top 0.50

# 영상 2 — 아래(잔디)만
python src/compare_crop.py --video outputs/test1/JPKYE9578.MOV \
    --model large --dtype fp32 --keep-top 0.45

# contrast 표
python src/measure_drone_contrast.py --video outputs/test1/IMG_E5065.MOV \
    --cut-top 0.25 --keep-top 0.50
```

| 항목 | 값 |
|---|---|
| GPU | NVIDIA RTX PRO 4500 Blackwell 32GB (sm_120) |
| Python / torch | 3.14.4 / 2.13.0+cu130 |
| 모델 | `depth-anything/Depth-Anything-V2-Large-hf` · 335.3M |
| 보간 / 컬러맵 | bilinear(align_corners=True) / Spectral_r + min-max — 공식 구현과 동일 |

`#dav3` 영상만 예외다 (예전에 받아둔 결과, 설정 미기록).

> 출력은 **상대 깊이**다. 값이 클수록 가깝고 절대 거리(m)가 아니다.
> 본문의 거리(0.3 m, 30 m 등)는 촬영 상황 추정치이며 모델 출력이 아니다.
{: .prompt-info }

## 원본 측정 파일 {#raw}

- [`IMG_E5065_large_fp32_cropcmp.csv`](/depth-anythingv2-test1/raw/IMG_E5065_large_fp32_cropcmp.csv) — 영상 1 렌더
- [`JPKYE9578_large_fp32_cropcmp.csv`](/depth-anythingv2-test1/raw/JPKYE9578_large_fp32_cropcmp.csv) — 영상 2 렌더
- [`IMG_E5065_large_fp32_drone_contrast_summary.csv`](/depth-anythingv2-test1/raw/IMG_E5065_large_fp32_drone_contrast_summary.csv) — contrast 표
- [`IMG_E5065_large_fp32_drone_contrast.csv`](/depth-anythingv2-test1/raw/IMG_E5065_large_fp32_drone_contrast.csv) — 256프레임 원본
- [`indoor_fp32_summary.md`](/depth-anythingv2-test1/raw/indoor_fp32_summary.md) — 실내 실시간 측정
