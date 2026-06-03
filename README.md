# LocalAI-miniPC iGPU Vulkan Acceleration Setup (v1.6)

[영문 가이드는 아래로 스크롤하세요. / Scroll down for the English guide.]

An optimized configuration and guide for deploying **LocalAI** with **Intel iGPU (UHD Graphics) Vulkan acceleration** inside a Proxmox LXC container on an Intel N95/N150 mini-PC. Achieve fast inference speeds on low-power hardware.

---

## 한국어 가이드 (Korean Guide)

### 1. v1.6 주요 개선 및 추가 사항
* **스킬북 및 종합 가이드 추가:** Proxmox LXC 구축, GPU 패스스루, Ollama 세팅 및 API 마이그레이션 기법을 다루는 문서 탑재.
* **인텔 내장 그래픽(iGPU) Vulkan 오프로딩:** LLM 추론 연산(GGUF 레이어)을 내장 UHD Graphics GPU로 이식하여 추론 속도를 극대화.
* **특권 컨테이너(`privileged: true`) 설정:** 중첩 가상화(Nested Virtualization) 하에서의 GPU 장치 접근 권한 오류를 차단.
* **기능 강제 환경변수 우회:** `/run/localai/capability` 파일 내부의 개행문자 버그(`vulkan\n`)를 우회하기 위해 `LOCALAI_FORCE_META_BACKEND_CAPABILITY=vulkan`을 강제 주입.
* **모델 템플릿 최적화:** Qwen 2.5 Instruct 모델의 한국어 답변 오작동 및 무한 루프 방지 정지어(stop token) 설정 완료.
* **동적 IP 디스커버리 및 포트 대응:** DHCP 환경의 동적 IP 스캔 및 Ollama 기본 포트(`11434`)와의 호환성 고려.
* **비전 모델(Moondream2) 정밀도 향상:** `moondream2-text-model-f16.gguf` (FP16 정밀도) 텍스트 가중치 모델로의 이식 완료.

---

### 2. ⚠️ [필수] 신규 유저 올인원 구축 가이드

이 저장소는 기존 LocalAI Docker 기반에서 **Ollama 단독 구동 체제**로 일원화 및 최적화되었습니다.
LXC 환경에서 처음 구축을 시작하는 분들은 헤매지 마시고 **가장 먼저 아래의 올인원 구축 가이드를 순서대로 진행**해 주시기 바랍니다.

> [!IMPORTANT]
> Proxmox LXC 템플릿 생성부터 GPU 패스스루 설정, Ollama 엔진 설치 및 가속 최적화까지 모든 과정이 A to Z로 담겨 있습니다.
> 👉 **[🖥️ SuperLLM LXC 신규 구축 가이드 문서 열기](./superllm_lxc_setup_guide.md)**

(위 가이드를 모두 마치신 후, 아래의 3번 API 호출 테스트로 넘어오시면 됩니다.)

---

### 3. 작동 검증 API 호출 테스트

LXC 내부 혹은 동등 네트워크 환경의 기기에서 아래 curl 명령을 통해 정상적인 한글 챗 완성 응답이 출력되는지 확인합니다:

```bash
curl -X POST http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:3b",
    "messages": [
      {"role": "system", "content": "너는 스마트홈 전문가야. 친절하고 간결하게 답변해줘."},
      {"role": "user", "content": "인텔 N150 미니 PC에서 너를 구동하는 데 성공했어! 축하 한마디 해줄래?"}
    ],
    "temperature": 0.7
  }'
```

---

### 4. 추가 기술 문서 및 스킬북 (Reference Skills)
본 저장소에는 미니 PC 기반의 AI 인프라 최적화를 돕기 위한 상세 스킬북 및 구축 문서들이 포함되어 있습니다:
* [Intel iGPU Vulkan 가속 설정 스킬북](file:///d:/Antigravity/localai-server/skills/igpu_vulkan_acceleration.md): Intel UHD Graphics 기반 하드웨어 가속 컴파일 및 Docker 상세 설정 가이드.
* [Ollama 기반 마이그레이션 스킬북](file:///d:/Antigravity/localai-server/skills/localai_migration.md): Ollama API 규격을 OpenAI 호환 API 규격으로 소스코드 수준에서 마이그레이션하는 개발 스킬 가이드.
* [SuperLLM LXC 신규 구축 가이드](file:///d:/Antigravity/localai-server/superllm_lxc_setup_guide.md): Proxmox 9.x 환경에서 우분투 24.04 컨테이너를 직접 구축하고 GPU 패스스루 권한 및 Ollama(11434 포트 리슨)를 설치하는 통합 매뉴얼.

---

### 5. 네트워크 최적화 (동적 IP 디스커버리)
* **동적 IP 디스커버리:** 미니 PC 환경에서 DHCP 할당에 의해 Ollama 서버의 IP가 변경되더라도, 에이전트 서비스가 네트워크 스캔 또는 mDNS(`superllm.local`)를 통해 자동으로 실시간 활성 서버 IP를 찾아 바인딩하는 기법을 권장합니다.

---
---

## English Guide

### 1. Key Features in v1.6
* **Detailed Skills & Setup Guides:** Added documentation for Proxmox LXC creation, GPU pass-through, Ollama configuration, and API migration.
* **Intel iGPU Vulkan Offloading:** Offloads LLM inference computations (GGUF layers) to the integrated UHD Graphics.
* **Privileged Container Support:** Added `privileged: true` to bypass nested GPU permission issues.
* **Forced Capability Overrides:** Configured `LOCALAI_FORCE_META_BACKEND_CAPABILITY=vulkan` to bypass the newline character bug (`vulkan\n`) inside `/run/localai/capability`.
* **Optimized Model Configs:** Corrected chat template parameters and stop tokens for Qwen 2.5 Instruct models.
* **Dynamic IP Discovery & Ports:** Adapts to dynamic DHCP IP allocation and maintains compatibility with Ollama's default port (`11434`).
* **Enhanced Vision Model (Moondream2):** Upgraded text model parameter weights to `moondream2-text-model-f16.gguf` for better image analysis.

---

### 2. ⚠️ [MUST READ] All-in-One Setup Guide

This repository has been optimized to exclusively use the **Ollama standalone engine** instead of the legacy LocalAI Docker setup.
If you are starting fresh, please follow the comprehensive setup guide below before proceeding to any other steps.

> [!IMPORTANT]
> The guide covers everything from creating a Proxmox LXC template to GPU pass-through, Ollama installation, and iGPU acceleration optimization.
> 👉 **[🖥️ Open SuperLLM LXC Setup Guide](./superllm_lxc_setup_guide.md)**

(Once you have completed the guide, proceed to step 3 below for API verification.)

---

### 3. API Verification

Test the completion API using `curl` inside the LXC container or from your local machine:

```bash
curl -X POST http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:3b",
    "messages": [
      {"role": "system", "content": "You are a smart home assistant. Answer shortly."},
      {"role": "user", "content": "Hello! Introduce yourself."}
    ],
    "temperature": 0.7
  }'
```

---

### 4. Reference Skill Books & Setup Guides
This repository includes specialized guides to help you optimize low-power AI infrastructure:
* [Intel iGPU Vulkan Acceleration Skill Book](file:///d:/Antigravity/localai-server/skills/igpu_vulkan_acceleration.md): A technical guide for GPU pass-through on PVE LXC and compiling Vulkan backends.
* [Ollama to LocalAI Migration Skill Book](file:///d:/Antigravity/localai-server/skills/localai_migration.md): Explains code-level migrations from Ollama APIs to OpenAI-compatible formats.
* [SuperLLM LXC Setup Guide](file:///d:/Antigravity/localai-server/superllm_lxc_setup_guide.md): Step-by-step instructions to create an Ubuntu 24.04 LXC container on PVE, pass through iGPU, and configure Ollama with standard port mappings (`11434`).

---

### 5. Network Optimization (Dynamic IP Discovery)
* **Dynamic IP Discovery:** Since local mini-PC IPs can change due to DHCP, it is recommended to implement mDNS (`superllm.local`) or active network discovery in agent codes to dynamically resolve the Ollama server IP.
