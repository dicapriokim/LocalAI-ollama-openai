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

### 2. 인프라 설정 (Proxmox 호스트 및 LXC)

인텔 UHD 내장 그래픽 자원을 Proxmox 호스트(`root@pve`)에서 LXC 컨테이너(`SuperLLM`) 내부로 패스스루해야 합니다.

1. Proxmox 호스트 OS 터미널에서 LXC 설정 파일을 편집합니다:
   ```bash
   nano /etc/pve/lxc/<컨테이너VMID>.conf
   ```
2. 파일 맨 하단에 다음 설정을 추가하고 저장합니다:
   ```text
   unprivileged: 0
   lxc.apparmor.profile: unconfined
   lxc.cgroup2.devices.allow: c 226:0 rwm
   lxc.cgroup2.devices.allow: c 226:128 rwm
   lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
   lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
   ```
3. Proxmox Web GUI의 **LXC 옵션(Options) -> Features** 메뉴에서 **Nesting**을 활성화한 뒤 컨테이너를 재부팅합니다.

---

### 3. 설치 및 실행 절차

1. 저장소를 클론하고 디렉토리로 이동합니다:
   ```bash
   git clone https://github.com/dicapriokim/LocalAI-miniPC.git
   cd LocalAI-miniPC/localai
   ```
2. Docker 컨테이너를 실행합니다:
   ```bash
   docker compose up -d
   ```
3. **모델 다운로드:** `./models` 폴더 아래로 GGUF 모델 가중치를 다운로드합니다:
   ```bash
   cd models
   # Qwen-1.5b Text Model
   wget -O qwen2.5-1.5b-instruct.gguf https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf
   # Qwen-3b Text Model (Required for HA_MCP)
   wget -O qwen2.5-3b-instruct.gguf https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/resolve/main/qwen2.5-3b-instruct-q4_k_m.gguf
   # Moondream2 Vision Model (Required for Matter QR)
   wget -O moondream2-text-model-f16.gguf https://huggingface.co/moondream/moondream2-gguf/resolve/main/moondream2-text-model-f16.gguf
   wget -O moondream2-mmproj-f16.gguf https://huggingface.co/moondream/moondream2-gguf/resolve/main/moondream2-mmproj-f16.gguf
   cd ..
   ```
4. **백엔드 설치 (필수):** 컨테이너 내부에 Vulkan 가속 연산을 담당할 `llama-cpp` 백엔드를 직접 설치합니다:
   ```bash
   docker exec -it local-ai /local-ai backends install llama-cpp
   ```
5. **서비스 재시작:** 새로 설치된 백엔드 구동 바이너리를 로드할 수 있도록 서비스를 재시작합니다:
   ```bash
   docker compose restart
   ```

---

### 4. 작동 검증 API 호출 테스트

LXC 내부 혹은 동등 네트워크 환경의 기기에서 아래 curl 명령을 통해 정상적인 한글 챗 완성 응답이 출력되는지 확인합니다:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-1.5b",
    "messages": [
      {"role": "system", "content": "너는 스마트홈 전문가야. 친절하고 간결하게 답변해줘."},
      {"role": "user", "content": "인텔 N95 미니 PC에서 너를 구동하는 데 성공했어! 축하 한마디 해줄래?"}
    ],
    "temperature": 0.7
  }'
```

---

### 5. 추가 기술 문서 및 스킬북 (Reference Skills)
본 저장소에는 미니 PC 기반의 AI 인프라 최적화를 돕기 위한 상세 스킬북 및 구축 문서들이 포함되어 있습니다:
* [Intel iGPU Vulkan 가속 설정 스킬북](file:///d:/Antigravity/localai-server/skills/igpu_vulkan_acceleration.md): Intel UHD Graphics 기반 하드웨어 가속 컴파일 및 Docker 상세 설정 가이드.
* [Ollama 기반 마이그레이션 스킬북](file:///d:/Antigravity/localai-server/skills/localai_migration.md): Ollama API 규격을 OpenAI 호환 API 규격으로 소스코드 수준에서 마이그레이션하는 개발 스킬 가이드.
* [SuperLLM LXC 신규 구축 가이드](file:///d:/Antigravity/localai-server/superllm_lxc_setup_guide.md): Proxmox 9.x 환경에서 우분투 24.04 컨테이너를 직접 구축하고 GPU 패스스루 권한 및 Ollama(11434 포트 리슨)를 설치하는 통합 매뉴얼.

---

### 6. 네트워크 최적화 (동적 IP 디스커버리 및 11434 포트 대응)
* **동적 IP 디스커버리:** 미니 PC 환경에서 DHCP 할당에 의해 LocalAI/Ollama 서버의 IP가 변경되더라도, 에이전트 서비스가 네트워크 스캔(예: `nmap` 또는 포트 탐색) 또는 mDNS(`superllm.local`)를 통해 자동으로 실시간 활성 서버 IP를 찾아 바인딩하는 기법을 권장합니다.
* **Ollama 호환 포트 (11434) 대응:** 기존 Ollama 연동 프로젝트가 `http://<서버_IP>:11434` 포트를 하드코딩하여 호출 중인 경우, LocalAI Docker Compose의 포트 포워딩을 `11434:8080`으로 매핑하거나, 호스트 시스템 단에서 포트 리다이렉션을 설정하여 API 변경 없이 매끄럽게 통신을 이식할 수 있습니다.

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

### 2. Infrastructure Setup (Proxmox Host & LXC)

Ensure the Intel UHD Graphics are passed through from the Proxmox Host (`root@pve`) to the LXC container (`SuperLLM`).

1. Edit your LXC configuration file on your Proxmox host:
   ```bash
   nano /etc/pve/lxc/<CONTAINER_VMID>.conf
   ```
2. Append the following parameters at the end of the file:
   ```text
   unprivileged: 0
   lxc.apparmor.profile: unconfined
   lxc.cgroup2.devices.allow: c 226:0 rwm
   lxc.cgroup2.devices.allow: c 226:128 rwm
   lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
   lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
   ```
3. Enable **Nesting** under **LXC Options -> Features** in the Proxmox Web GUI, and reboot the LXC container.

---

### 3. Installation & Run

1. Clone the repository and navigate to the directory:
   ```bash
   git clone https://github.com/dicapriokim/LocalAI-miniPC.git
   cd LocalAI-miniPC/localai
   ```
2. Launch the LocalAI Docker container:
   ```bash
   docker compose up -d
   ```
3. **Download Model:** Download the GGUF model into the `./models` folder:
   ```bash
   cd models
   # Qwen-1.5b Text Model
   wget -O qwen2.5-1.5b-instruct.gguf https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf
   # Qwen-3b Text Model (Required for HA_MCP)
   wget -O qwen2.5-3b-instruct.gguf https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/resolve/main/qwen2.5-3b-instruct-q4_k_m.gguf
   # Moondream2 Vision Model (Required for Matter QR)
   wget -O moondream2-text-model-f16.gguf https://huggingface.co/moondream/moondream2-gguf/resolve/main/moondream2-text-model-f16.gguf
   wget -O moondream2-mmproj-f16.gguf https://huggingface.co/moondream/moondream2-gguf/resolve/main/moondream2-mmproj-f16.gguf
   cd ..
   ```
4. **Install Backend (CRITICAL):** Manually install the `llama-cpp` backend inside the container:
   ```bash
   docker exec -it local-ai /local-ai backends install llama-cpp
   ```
5. **Restart Container:** Restart the container to register the newly installed backend:
   ```bash
   docker compose restart
   ```

---

### 4. API Verification

Test the completion API using `curl` inside the LXC container or from your local machine:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-1.5b",
    "messages": [
      {"role": "system", "content": "You are a smart home assistant. Answer shortly."},
      {"role": "user", "content": "Hello! Introduce yourself."}
    ],
    "temperature": 0.7
  }'
```

---

### 5. Reference Skill Books & Setup Guides
This repository includes specialized guides to help you optimize low-power AI infrastructure:
* [Intel iGPU Vulkan Acceleration Skill Book](file:///d:/Antigravity/localai-server/skills/igpu_vulkan_acceleration.md): A technical guide for GPU pass-through on PVE LXC and compiling Vulkan backends.
* [Ollama to LocalAI Migration Skill Book](file:///d:/Antigravity/localai-server/skills/localai_migration.md): Explains code-level migrations from Ollama APIs to OpenAI-compatible formats.
* [SuperLLM LXC Setup Guide](file:///d:/Antigravity/localai-server/superllm_lxc_setup_guide.md): Step-by-step instructions to create an Ubuntu 24.04 LXC container on PVE, pass through iGPU, and configure Ollama with standard port mappings (`11434`).

---

### 6. Network Optimization (Dynamic IP Discovery & Port 11434 Compatibility)
* **Dynamic IP Discovery:** Since local mini-PC IPs can change due to DHCP, it is recommended to implement mDNS (`superllm.local`) or active network discovery (e.g., scanning ports) in agent codes to dynamically resolve the server IP.
* **Ollama Port 11434 Compatibility:** For existing integrations expecting Ollama on port `11434`, you can map LocalAI's external port to `11434` (e.g., `11434:8080` in Docker Compose) or use host-level port forwarding to ensure seamless API transition.
