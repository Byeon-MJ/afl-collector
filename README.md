# FAS Data Collectors — 모바일 실행 가이드

## 개요

Face Anti-Spoofing 데이터 수집 Web PWA 모음입니다.
모바일 브라우저에서 직접 실행하며, 클라이언트에서 완결됩니다.

| Collector | 파일 | 용도 | URL |
|-----------|------|------|-----|
| **AFL** | `index.html` | Flash 반사 패턴 수집 (AE/AWB 분석) | [열기](https://audin1.github.io/afl-collector/) |
| **PPLD** | `ppld_collector.html` | 원근 시차 (Parallax) 데이터 수집 | [열기](https://audin1.github.io/afl-collector/ppld_collector.html) |

### AFL (Adaptive Flash Liveness)
화면 flash → 카메라 캡처 → ZIP 다운로드. AE/AWB lock 분석 포함.

### PPLD (Perspective Parallax Liveness Detection)
얼굴 접근 동작 중 원근 왜곡을 측정하여 3D/2D를 구별합니다.
- **Tier 1**: Face bbox area growth rate, sub-bbox differential (center vs edge)
- **Tier 2**: MediaPipe 478-point landmarks — FNUS, CDS (Procrustes), DMS, SOS
- 실시간 HUD 오버레이 + 접근 가이드 + 자동 종료 + ZIP 다운로드

---

## 요구 사항

| 항목 | 최소 | 권장 |
|------|------|------|
| Android | Chrome 89+ | Chrome 120+ |
| iOS | Safari 17+ (iOS 17) | Safari 17.4+ |
| 카메라 | 전면 720p | 전면 1080p |
| 저장 공간 | 세션당 ~5MB | 500MB 이상 |

> **iOS 제한**: Safari는 `ImageCapture` API와 AE/AWB lock을 지원하지 않습니다. Canvas fallback과 delta 기반 보정으로 동작하지만, Android Chrome이 최적의 환경입니다.

---

## 실행 방법

### 방법 1: 로컬 서버 (같은 Wi-Fi)

개발 PC에서 HTTPS 서버를 띄우고 모바일로 접속합니다.
카메라 API는 HTTPS 또는 localhost에서만 동작합니다.

**1) Python (가장 간단)**

```bash
cd /home/audin/sim/DGUA/active_liveness/web_collector

# 자체 서명 인증서 생성 (최초 1회)
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj '/CN=localhost'

# HTTPS 서버 시작
python3 -c "
import ssl, http.server
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain('cert.pem', 'key.pem')
server = http.server.HTTPServer(('0.0.0.0', 8443), http.server.SimpleHTTPRequestHandler)
server.socket = ctx.wrap_socket(server.socket, server_side=True)
print('Serving on https://0.0.0.0:8443')
server.serve_forever()
"
```

**2) Node.js**

```bash
npx http-server -S -C cert.pem -K key.pem -p 8443
```

**3) 모바일에서 접속**

```
https://<PC의_IP>:8443/index.html
```

PC IP 확인: `ip addr show | grep 'inet '` 또는 `hostname -I`

> 자체 서명 인증서 경고가 나옵니다 → "고급" → "안전하지 않음 계속" 선택

### 방법 2: ngrok (외부 네트워크)

```bash
cd /home/audin/sim/DGUA/active_liveness/web_collector

# 일반 HTTP 서버 시작
python3 -m http.server 8080

# 다른 터미널에서 ngrok으로 HTTPS 터널
ngrok http 8080
```

ngrok이 제공하는 `https://xxxxx.ngrok-free.app` URL을 모바일에서 접속합니다.

### 방법 3: GitHub Pages (가장 편리)

```bash
# 별도 저장소 또는 브랜치에 index.html 배포
# GitHub Pages는 자동 HTTPS 제공
```

1. GitHub에 `afl-collector` 저장소 생성
2. `index.html` 업로드
3. Settings → Pages → Deploy from branch
4. `https://<username>.github.io/afl-collector/` 로 접속

---

## 수집 절차

### 1. 설정 화면

| 설정 | Live 수집 | Spoof 수집 | 설명 |
|------|-----------|------------|------|
| 수집 유형 | `Live (실제 얼굴)` | `Spoof: Screen Replay` 등 | |
| Flash 프레임 수 | 4 (기본) | 4 (기본) | |
| Flash 지속 시간 | 150ms (기본) | 150ms (기본) | 길수록 멀티프레임 수 증가 |
| **Flash 강도 (Alpha)** | **Auto 또는 T2** | **Auto 또는 T2** | **아래 표 참조** |
| 세션 수 | 10~50 | 10~50 | |
| AE/AWB 분석 모드 | 체크 권장 | 체크 권장 | |

#### Flash 강도 티어

| 모드 | Alpha | 설명 | 용도 |
|------|-------|------|------|
| **Auto** | 0.04~0.25 | 주변광 측정 후 자동 결정 | 프로덕션 권장 |
| T1: Invisible | 0.04 | 거의 인지 불가 | 안전성 최우선 (멀티프레임 필수) |
| T2: Subtle | 0.10 | 약간 인지 가능 | **프로덕션 기본값** |
| T3: Moderate | 0.20 | 명확히 보이나 편안함 | 밝은 환경 폴백 |
| T4: Strong | 1.00 | 전체 화면 플래시 | 시험/연구 전용 |
| Custom | 0.01~1.00 | 슬라이더로 직접 설정 | 세밀 실험용 |

> **광과민성 안전**: Alpha ≤ 0.10에서는 WCAG 2.1 SC 2.3.1 "10% 휘도 변화" 미만이므로 3-flash 규칙이 적용되지 않습니다. T1/T2가 프로덕션에 안전합니다.

"카메라 시작" 버튼을 누르면 카메라 권한을 요청합니다.

### 2. 카메라 기능 확인

설정 화면 하단에 디바이스 카메라 지원 정보가 표시됩니다:

```
AE Lock: ✓ 지원          ← Android Chrome에서 주로 지원
AWB Lock: ✓ 지원
해상도: 1280×720
FPS: 30
ImageCapture API: ✓      ← iOS Safari에서는 ✗
```

### 3. 캡처

1. 얼굴을 화면 가이드(점선 원) 안에 맞춤
2. **"Flash 캡처 시작"** 버튼 탭
3. **주변광 측정** — 3프레임을 캡처하여 밝기 분석 → Auto 모드 시 alpha 자동 결정
4. 카메라 프리뷰 위에 **반투명 색상 오버레이**가 표시되며 프레임 캡처 (T1~T3)
   - Flash당 **최대 4프레임** 자동 캡처 (멀티프레임, flash 지속시간에 따라)
5. AE/AWB 분석 모드면 동일 시퀀스를 한 번 더 실행 (AE 해제 상태)
6. 세션 완료 → 다음 세션 버튼 탭 반복

> **T4(Strong)에서만** 기존처럼 전체 화면이 색상으로 채워집니다. T1~T3에서는 카메라 프리뷰 위에 살짝 색이 덮이는 수준입니다.

### 4. 다운로드

모든 세션 완료 후 **"데이터 다운로드 (ZIP)"** 버튼으로 저장합니다.

---

## Spoof 데이터 수집 방법

### Screen Replay (화면 재생 공격)

1. **기기 A** (공격 재생용): 실제 얼굴 영상/사진을 전체 화면 표시
2. **기기 B** (수집용): AFL Collector를 실행하고, 기기 A의 화면을 카메라로 촬영
3. 수집 유형: `Spoof: Screen Replay` 선택

```
[기기 B: AFL Collector] ──카메라──▶ [기기 A: 얼굴 영상 재생 중]
```

### Print 공격

1. 실제 얼굴 사진을 A4 컬러 인쇄
2. AFL Collector를 실행하고, 인쇄물을 카메라 앞에 배치
3. 수집 유형: `Spoof: Print` 선택

### 3D Mask

1. 3D 마스크를 착용하거나 카메라 앞에 배치
2. 수집 유형: `Spoof: 3D Mask` 선택

---

## 출력 데이터 구조

```
afl_data_live_2026-03-19.zip
└── live/
    └── session_20260319_0001/
        ├── metadata.json              # 전체 메타데이터 (alpha, ambient, multiframe 포함)
        ├── baseline.png               # Pass 1: primary 프레임 (AE locked)
        ├── flash_001.png              # Pass 1: Red flash
        ├── flash_002.png ~ 00N.png
        ├── calibration.png            # Pass 1: White
        ├── ae_auto_baseline.png       # Pass 2: AE auto (분석 모드 전용)
        ├── ae_auto_flash_001.png ~ 00N.png
        ├── ae_auto_calibration.png
        └── multiframe/               # 멀티프레임 (Pass 1 + Pass 2)
            ├── baseline_f0.png        # Pass 1 멀티프레임
            ├── baseline_f1.png ~ fN.png
            ├── flash_001_f0.png ~ fN.png
            ├── ae_auto_baseline_f0.png  # Pass 2 멀티프레임
            ├── ae_auto_flash_001_f0.png ~ fN.png
            └── ...
```

### metadata.json 주요 필드

```json
{
  "challengeId": "a1b2c3...",
  "collectType": "live",
  "flashAlphaMode": "auto",
  "flashAlpha": 0.10,
  "ambientMeasurement": {
    "avgBrightness": 95.3,
    "selectedAlpha": 0.10,
    "mode": "auto"
  },
  "aeAnalysisMode": true,
  "aeLocked": true,
  "awbLocked": true,
  "flashDuration": 150,
  "totalDurationMs": 4200,
  "frames": [
    {
      "frameId": 0,
      "type": "baseline",
      "colorRGB": [0, 0, 0],
      "flashAlpha": 0.10,
      "latencyMs": 62.5,
      "brightness": 45.2,
      "meanR": 42.1, "meanG": 46.8, "meanB": 46.7,
      "framesPerFlash": 3,
      "multiFrameBrightness": [
        {"brightness": 44.8, "meanR": 41.9, "meanG": 46.2, "meanB": 46.3, "latencyMs": 45.2},
        {"brightness": 45.2, "meanR": 42.1, "meanG": 46.8, "meanB": 46.7, "latencyMs": 78.4},
        {"brightness": 45.6, "meanR": 42.3, "meanG": 47.1, "meanB": 47.4, "latencyMs": 112.1}
      ]
    }
  ],
  "aeAutoFrames": [ "... (동일 구조, Pass 2)" ],
  "aeResponseCurve": [
    {
      "frameId": 0,
      "colorRGB": [0, 0, 0],
      "lockedBrightness": 45.2,
      "autoBrightness": 48.7,
      "brightnessDelta": 3.5,
      "rgbDelta": [2.1, 4.2, 4.1]
    }
  ],
  "aeResponseSummary": {
    "meanDelta": 5.3,
    "maxDelta": 12.8,
    "minDelta": -1.2,
    "stdDelta": 4.1
  }
}
```

---

## 디바이스별 참고 사항

### Android (Chrome)

- AE/AWB lock 지원 (대부분의 기기)
- ImageCapture API 지원 → 정확한 타이밍의 프레임 캡처
- 화면 밝기를 최대로 설정하세요 (flash 효과 극대화)
- 자동 화면 꺼짐 비활성화 권장: 설정 → 디스플레이 → 화면 자동 꺼짐 → 10분

### iOS (Safari)

- AE/AWB lock 미지원 → delta 기반 보정으로 동작
- ImageCapture API 미지원 → Canvas fallback 사용
- **iOS 17 이상 필수** (이전 버전은 `getCapabilities()` 없음)
- 화면 밝기 최대 + True Tone 비활성화 권장
- 저전력 모드 비활성화 (프레임 레이트 제한 방지)

### Samsung Galaxy (One UI)

- Chrome에서 가장 안정적
- Samsung Internet도 동작하나 Chrome 권장
- "게임 부스터"가 활성화되어 있으면 성능 제한될 수 있음 → 비활성화

---

## 권장 수집 규모

| 단계 | Live | Screen Replay | Print | 3D Mask |
|------|------|---------------|-------|---------|
| Pilot (검증용) | 20 | 20 | 10 | - |
| Training (v1) | 200 | 200 | 100 | 50 |
| Production | 1000+ | 1000+ | 500+ | 200+ |

- 다양한 기기에서 수집 (최소 3종 이상)
- 다양한 조명 환경 (실내, 실외, 야간)
- AE/AWB 분석 모드 항상 활성화 권장

---

## 문제 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 카메라 접근 거부 | HTTP 접속 | HTTPS로 접속 필수 |
| AE Lock 미지원 | iOS Safari / 구형 브라우저 | 정상 — delta 보정으로 동작 |
| 화면 flash가 안 보임 (T1/T2) | 정상 동작 — 저강도 모드 | alpha를 높이거나 T3/T4로 변경 |
| 화면 flash가 안 보임 (T4) | 화면 밝기 낮음 | 밝기 최대로 설정 |
| ZIP 다운로드 안 됨 | 저장 공간 부족 | 기기 공간 확보 |
| 프레임이 어둡게 캡처 | AE가 flash에 반응 | flash 지속 시간을 100ms로 줄임 |
| 느린 캡처 속도 | 기기 성능 제한 | flash 프레임 수를 3으로 줄임 |
