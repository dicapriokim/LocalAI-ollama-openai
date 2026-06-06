# 🖥️ SuperLLM LXC 신규 구축 및 Ollama 설정 가이드 (Ubuntu 24.04 LTS)

본 문서는 새로운 Proxmox VE 9.x 호스트(`<PVE_HOST_IP>`, `<PVE_NODE_NAME>`) 위에 새로운 LXC 컨테이너를 직접 생성하고, Ollama와 필수 AI 모델(`qwen2.5:3b`, `moondream`)을 처음부터 완벽하게 설치 및 설정하는 단계별 설명서입니다.

[영문 가이드는 아래로 스크롤하세요. / Scroll down for the English guide.]

---

## 🛠️ [Step 1] Proxmox VE에서 신규 LXC 컨테이너 생성

### 1. Ubuntu 24.04 템플릿 다운로드
1. 웹 브라우저를 열어 새로운 PVE 관리 페이지(`https://<PVE_HOST_IP>:8006`)에 로그인합니다.
2. 좌측 네비게이션 트리에서 **`<PVE_NODE_NAME>` 노드** ➔ **`local` (또는 local-lvm) 스토리지**를 클릭합니다.
3. **`CT Templates` (컨테이너 템플릿)** 메뉴를 선택하고 **`Templates`** 버튼을 누릅니다.
4. 검색창에 `ubuntu`를 입력한 뒤 **`ubuntu-24.04-standard_..._amd64.tar.zst`**를 선택하고 **`Download`**를 클릭합니다. (다운로드가 완료될 때까지 대기)

### 2. LXC 컨테이너 생성 마법사 실행
웹 대시보드 우측 상단의 **`Create CT` (컨테이너 생성)** 버튼을 클릭하여 아래 사양대로 설정을 진행합니다:

* **General (일반)**
  * **Node**: `<PVE_NODE_NAME>`
  * **CT ID**: 비어있는 임의의 ID 지정 (예: `100` 이상)
  * **Hostname**: `SuperLLM`
  * **Password**: 루트 패스워드 설정 및 확인 입력 (예: 사용자 지정 비밀번호)
  * **Unprivileged container (비특권 컨테이너)**: **체크 해제** (LXC 내부 Docker 연동 및 마운트 권한 꼬임 방지를 위해 특권 컨테이너로 생성)
* **Template (템플릿)**
  * **Storage**: 템플릿이 저장된 스토리지 선택
  * **Template**: 다운로드한 `ubuntu-24.04-standard` 선택
* **Root Disk (루트 디스크)**
  * **Storage**: 설치할 스토리지 선택 (예: `local-lvm` 등)
  * **Disk size (GiB)**: **`20` ~ `30` GiB** 권장 (Ollama 모델 보관 및 여유 공간 확보용)
* **CPU**
  * **Cores**: **`4`** 지정 (Intel N150의 4코어 전체를 할당하여 추론 속도 최적화)
* **Memory (메모리)**
  * **Memory (MiB)**: **`4096` ~ `8192` MiB** (최소 4GB, 가능하면 8GB 이상 권장)
  * **Swap (MiB)**: `512` ~ `1024` MiB
* **Network (네트워크)**
  * **Bridge**: `vmbr0`
  * **IPv4**: `Static`
  * **IPv4/CIDR**: **`<LXC_IP>/24`** 입력 (기존 프로젝트와의 호환성을 위해 할당된 IP 권장)
  * **Gateway (IPv4)**: `<GATEWAY_IP>`
* **DNS**
  * 기본 설정 유지 (Use host settings)
* **Confirm (확인)**
  * 설정을 최종 검토한 뒤 **`Finish`**를 눌러 컨테이너를 생성합니다.

### 3. Docker 구동을 위한 필수 추가 설정 (중요)
LXC 내부에서 Docker(Mail-Automator 등)가 정상 구동되려면 생성 직후 반드시 아래 기능을 활성화해야 합니다:
1. Proxmox 웹 UI 좌측 메뉴에서 생성된 **`SuperLLM` (LXC 컨테이너)**를 클릭합니다.
2. **`Resources` 및 `Options` (옵션)** ➔ **`Features` (기능)** 메뉴를 더블 클릭합니다.
3. **`keyctl`**과 **`nesting`** 두 항목을 모두 체크하여 활성화한 뒤 **`OK`**를 누릅니다.
   *(※ 이 설정을 누락하면 LXC 내부에서 Docker 데몬이 초기화 실패로 실행되지 않습니다.)*

### 4. Intel iGPU 하드웨어 가속(GPU 패스스루) 추가 설정 (성능 가속용 권장)
미니 PC N150의 내장 그래픽(iGPU UHD Graphics) 연산 자원을 컨테이너 내부로 넘겨 Ollama 추론 속도를 가속화하려면 다음 설정을 주입합니다:
1. Proxmox VE 호스트 터미널(`root@<PVE_NODE_NAME>`)에 로그인합니다.
2. 해당 `SuperLLM` 컨테이너의 설정 파일(예: VMID가 `103`인 경우 `/etc/pve/lxc/103.conf`)을 편집합니다:
   ```bash
   nano /etc/pve/lxc/<CT_ID>.conf
   ```
3. 파일 최하단에 다음의 디바이스 맵핑 및 보안 프로필 규격을 주입하고 저장합니다:
   ```text
   unprivileged: 0
   lxc.apparmor.profile: unconfined
   lxc.cgroup2.devices.allow: c 226:0 rwm
   lxc.cgroup2.devices.allow: c 226:128 rwm
   lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
   lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
   ```
4. 설정을 반영한 후 LXC 컨테이너를 완전히 **재부팅(Reboot)**하면, 컨테이너 내에서 그래픽 장치(`/dev/dri/renderD128`)를 완벽히 인지하여 Ollama가 하드웨어 GPU 가속을 사용해 비약적으로 가속화된 답변 속도를 보여주게 됩니다.

---

## ⚙️ [Step 2] 컨테이너 기본 패키지 및 종속성 구성

LXC 생성이 완료되면 생성된 컨테이너를 클릭하고 **`Start`**를 눌러 구동시킨 뒤, **`Console`** 메뉴 또는 터미널(SSH)을 통해 루트 계정으로 진입하여 다음 명령어를 순서대로 실행합니다.

```bash
# 1. 패키지 인덱스 업데이트 및 업그레이드
apt update && apt upgrade -y

# 2. 필수 빌드/네트워크 유틸리티 및 SSH 서버 설치 (zstd는 ollama 필수)
apt install -y curl zstd net-tools git avahi-daemon openssh-server

# 3. 외부 접속용 root SSH 로그인 및 비밀번호 인증 설정 허용
sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/g' /etc/ssh/sshd_config
systemctl restart sshd
systemctl enable ssh

# 4. Docker Engine 자동 설치 및 구동 (Mail-Automator 기동용)
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
```

* `avahi-daemon`은 로컬망에서 IP 주소가 바뀌더라도 `superllm.local` 호스트명으로 자동 바인딩될 수 있게 돕는 유연한 mDNS 유틸리티입니다.

---

## 🦙 [Step 3] Ollama 설치 및 외부 바인딩(0.0.0.0) 설정

Ollama는 기본적으로 로컬호스트(`127.0.0.1`)에서만 요청을 수신하도록 기본값이 잡혀 있어, 프로젝트들이 통신하려면 **모든 IP 대역(`0.0.0.0`)에서 리슨하도록 환경 변수를 재설정**해야 합니다.

### 1. Ollama 엔진 원클릭 설치
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 2. OLLAMA_HOST 환경변수 오버라이드 적용
외부(Home Assistant, Mail-Automator 등)에서 컨테이너 포트 `11434`로 직접 질의할 수 있도록 systemd 서비스 유닛 설정을 오버라이드합니다.

```bash
# systemd 서비스 설정용 Drop-in 디렉토리 생성
mkdir -p /etc/systemd/system/ollama.service.d

# 오버라이드 설정 파일 주입
cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_IGPU_ENABLE=1"
EOF

# 변경 사항 데몬 로드 및 서비스 재시작
systemctl daemon-reload
systemctl restart ollama
```

### 3. 정상 동작 및 바인딩 상태 검증
```bash
# 포트 11434가 *:11434 또는 0.0.0.0:11434 상태로 리슨 중인지 확인
netstat -tlnp | grep 11434
```

---

## 📥 [Step 4] 필수 AI 모델 다운로드 (qwen2.5:3b & moondream)

> [!NOTE]
> **양자화 모델 다운로드 안내**: 
> Ollama에서 모델명 뒤에 별도의 사양 태그를 붙이지 않고 기본 명령어로 다운로드하면, **기본적으로 4비트 양자화(Q4_K_M)가 기본 적용된 경량화 모델**을 가져오게 됩니다. 
> 원본 FP16 모델 대비 성능 저하를 최소화하면서 용량과 CPU 연산량을 극적으로 낮춘 상태(1.7~1.9GB 수준)로 내려받기 때문에, 저전력 N150/N95 CPU 환경에서 가장 이상적인 속도와 정확도를 보장합니다. 따라서 아래 기본 명령을 그대로 실행하시면 됩니다.

Ollama 서버가 정상 구동되었다면 다음 명령어를 실행하여 텍스트 추론용 모델과 비전 처리용 모델을 안전하게 로컬 저장소로 다운로드(Pull)합니다.

```bash
# 1. 한국어 질의 및 MCP 도구 판단용 경량 모델 풀링 (약 1.9 GB)
ollama pull qwen2.5:3b

# 2. QR 스캔 OCR 및 이미지 분석용 멀티모달 비전 모델 풀링 (약 1.7 GB)
ollama pull moondream

# 3. 받아진 모델 목록 확인
ollama list
```

> [!TIP]
> **왜 qwen2.5:1.5b 대신 qwen2.5:3b를 사용하나요?**
> * **지능 및 정확도 격차**: `1.5b` 모델은 매개변수가 너무 적어 스마트홈 기기 제어 흐름(도구 이름 및 파라미터 판단) 판단력이 떨어지고 한국어 문맥 왜곡 현상이 잦습니다. 
> * **하드웨어 가용성 보장**: Intel N150은 TDP 6W로 낮지만 Gracemont 아키텍처 기반의 강력한 x86 4코어 CPU이므로, `qwen2.5:3b` (실 사용량 약 2GB) 모델을 쾌적하게 돌릴 수 있는 충분한 연산력을 냅니다.
> * **자원 및 메모리 최적화**: 1.5B와 3B를 동시에 띄워두는 것보다, 가장 지능이 뛰어난 **`qwen2.5:3b` 하나로 모델을 일원화**하여 Ollama RAM 캐싱을 독점시키는 것이 자원 관리와 전체 시스템 반응성 측면에서 훨씬 유리합니다.

---

## 🧪 [Step 5] API 통신 확인 최종 테스트

LXC 내부 콘솔 혹은 외부 PC의 셸에서 아래 curl 명령을 보내 Ollama의 OpenAI 규격 chat completions API가 정상적으로 JSON 결과를 돌려주는지 최종 검증합니다.

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:3b",
    "messages": [
      {"role": "user", "content": "안녕하세요! 짧게 한 단어로 답해주세요."}
    ],
    "max_tokens": 10
  }'
```

정상적으로 `{"choices": [{"message": {"content": "..."}}]}` 형태의 JSON 응답이 반환되면 AI 통합 서버 구축이 완전히 끝난 것입니다!

---

## 🏎️ [Step 6] iGPU 하드웨어 가속 작동 및 성능 향상 검증 방법

LXC에 패스스루한 Intel N150 내장 그래픽 카드가 실제로 활성화되어 작동 중인지, 그리고 그로 인해 얼마나 속도가 빨라졌는지 측정하는 방법입니다.

### 1. LXC 컨테이너 내부 GPU 장치 인식 상태 확인
컨테이너 내부 셸에서 아래 명령을 실행합니다:
```bash
# /dev/dri 디렉토리 및 renderD128 디바이스 노출 여부 확인
ls -la /dev/dri
```
* **정상 결과**: `card0` 및 `renderD128` 장치 파일이 깨끗하게 출력되어야 합니다.

### 2. Ollama 엔진의 GPU 감지 로그 확인
Ollama가 기동될 때 그래픽 드라이버 자원을 제대로 물고 일어났는지 systemd 로그에서 확인합니다:
```bash
journalctl -u ollama --no-pager -n 50
```
* **정상 결과**: 로그 내용 중 **`Vulkan`**, **`UHD Graphics`** 또는 **`initialization`** 관련 감지 성공 로그가 포함되어야 합니다. 만약 장치 접근에 문제가 있다면 `CPU-only` 또는 `fallback` 오류가 출력됩니다.

### 3. 실시간 GPU 부하 모니터링 (성능 체크)
인텔 GPU 전용 실시간 모니터링 도구(`intel_gpu_top`)를 사용하여 추론 연산이 발생할 때 GPU 부하율(3D/Render %)이 치솟는지 확인합니다.

```bash
# 1. GPU 모니터링 도구 설치 (컨테이너 내부 또는 PVE 호스트에서 실행)
apt install -y intel-gpu-tools

# 2. 실시간 모니터러 실행
intel_gpu_top
```
* **테스트 진행**: 모니터링을 켜둔 상태에서 다른 터미널 창을 열어 아래와 같이 조금 무거운 질문을 봇에 전송합니다:
  ```bash
  curl http://localhost:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "qwen2.5:3b",
      "messages": [{"role": "user", "content": "우주와 블랙홀의 탄생 과정에 대해 자세히 설명해줘."}]
    }'
  ```
* **가속 작동 판정**: 질문을 던진 순간 `intel_gpu_top` 대시보드의 **`Render/3D` 엔진 부하율이 50%~100%** 근처로 크게 흔들리며 연산이 시작된다면, iGPU 하드웨어 가속이 100% 정상 작동 중인 것입니다.

### 4. 추론 속도(Tokens Per Second) 성능 비교 체감
Ollama의 추론 로그 타이밍 결과는 `journalctl -u ollama` 또는 Ollama CLI 직접 실행 시 성능 통계치로 확인 가능합니다:
* **기존 CPU 전용 모드**: 대략 **`5~8 tokens per second`** 전후로 생성됩니다.
* **iGPU 가속 모드**: 대략 **`12~18 tokens per second` 이상**으로 속도가 최소 **1.5배~2배 이상 향상**되어 문장이 훨씬 빠르게 타이핑되듯이 완성됩니다.

### 5. 문제 해결 (트러블슈팅): 여전히 CPU로만 장치를 인식하고 VRAM이 "0 B"일 때
패스스루 설정을 했음에도 Ollama가 내장 그래픽 자원을 로드하지 못하고 CPU로 폴백할 경우, 다음 조치 명령을 LXC 컨테이너 내부 터미널에서 즉시 실행합니다.

#### 조치 1: 컨테이너 내부의 디바이스 파일 권한(Permission) 개방 및 그룹 소속 추가
LXC 내부의 `ollama` 사용자가 호스트에서 넘어온 디바이스 드라이버 파일에 접근할 권한이 막혀있어 감지하지 못하는 현상입니다. 

특히 `renderD128`의 그룹 권한이 `video`가 아닌 `clock` 등으로 잡혀 있는 경우, 다음 명령으로 권한 개방 및 그룹 추가를 진행합니다:
```bash
# 1. 임시로 그래픽 장치 노드 전체 읽기/쓰기 권한 부여
chmod 666 /dev/dri/card0
chmod 666 /dev/dri/renderD128

# 2. ollama 서비스 구동 계정에 그래픽 장치 관련 그룹 권한 강제 추가
usermod -aG video ollama
usermod -aG clock ollama
```

#### 조치 2: 컨테이너 내부 인텔 GPU 및 Vulkan/Mesa 드라이버 패키지 누락 해결
컨테이너 내부에 하드웨어 가속용 런타임 라이브러리가 없어 드라이버 바인딩에 실패하는 경우입니다. 다음 필수 그래픽 라이브러리 패키지들을 컨테이너 내에 설치합니다:
```bash
# LXC 컨테이너 내부에서 실행
apt update
apt install -y mesa-vulkan-drivers va-driver-all ocl-icd-libopencl1 intel-opencl-icd clinfo
```

#### 조치 3: Ollama 서비스 재시작 및 동작 검증
위 조치들을 취한 후, 변경사항을 적용하기 위해 Ollama 엔진 데몬을 재기동합니다:
```bash
systemctl restart ollama
```
재기동 후 `ollama ps`를 쳐서 **`PROCESSOR`가 `100% GPU`**로 정상 표시되며 하드웨어 가속이 활성화되었는지 다시 체크합니다.

---
---

## English Guide

# 🖥️ SuperLLM LXC Setup & Ollama Configuration Guide (Ubuntu 24.04 LTS)

This document provides a step-by-step guide to creating a new LXC container on a new Proxmox VE 9.x host (`<PVE_HOST_IP>`, `<PVE_NODE_NAME>`), and perfectly installing and configuring Ollama with essential AI models (`qwen2.5:3b`, `moondream`) from scratch.

---

## 🛠️ [Step 1] Create a New LXC Container in Proxmox VE

### 1. Download Ubuntu 24.04 Template
1. Open a web browser and log in to your PVE management page (`https://<PVE_HOST_IP>:8006`).
2. In the left navigation tree, click on your **`<PVE_NODE_NAME>` node** ➔ **`local` (or local-lvm) storage**.
3. Select the **`CT Templates`** menu and click the **`Templates`** button.
4. Search for `ubuntu`, select **`ubuntu-24.04-standard_..._amd64.tar.zst`**, and click **`Download`**. (Wait until finished)

### 2. Run the LXC Creation Wizard
Click the **`Create CT`** button at the top right of the web dashboard and proceed with the following settings:

* **General**
  * **Node**: `<PVE_NODE_NAME>`
  * **CT ID**: Assign any empty ID (e.g., `100` or above)
  * **Hostname**: `SuperLLM`
  * **Password**: Set and confirm a root password
  * **Unprivileged container**: **Uncheck** (Create as a privileged container to prevent Docker mounting permission issues)
* **Template**
  * **Storage**: Select where the template is stored
  * **Template**: Choose the downloaded `ubuntu-24.04-standard`
* **Root Disk**
  * **Storage**: Select installation storage (e.g., `local-lvm`)
  * **Disk size (GiB)**: **`20` to `30` GiB** recommended (for storing Ollama models)
* **CPU**
  * **Cores**: **`4`** (Allocate all 4 cores of the Intel N150 to optimize inference speed)
* **Memory**
  * **Memory (MiB)**: **`4096` to `8192` MiB** (At least 4GB, 8GB+ recommended)
  * **Swap (MiB)**: `512` to `1024` MiB
* **Network**
  * **Bridge**: `vmbr0`
  * **IPv4**: `Static`
  * **IPv4/CIDR**: Enter **`<LXC_IP>/24`**
  * **Gateway (IPv4)**: `<GATEWAY_IP>`
* **DNS**
  * Use host settings
* **Confirm**
  * Review the settings and click **`Finish`** to create the container.

### 3. Essential Settings for Docker (Important)
For Docker (e.g., Mail-Automator) to run normally inside LXC, enable the following features immediately after creation:
1. Click the created **`SuperLLM` (LXC Container)** in the left menu.
2. Double-click the **`Features`** menu under **`Options`**.
3. Check both **`keyctl`** and **`nesting`**, then click **`OK`**.
   *(※ If omitted, the Docker daemon will fail to initialize inside the LXC.)*

### 4. Intel iGPU Hardware Acceleration (GPU Pass-through) Settings (Recommended)
To pass the integrated graphics (iGPU UHD Graphics) resources of the N150 into the container for Ollama acceleration:
1. Log in to the Proxmox VE host terminal (`root@<PVE_NODE_NAME>`).
2. Edit the container's configuration file (e.g., if VMID is `103`, `/etc/pve/lxc/103.conf`):
   ```bash
   nano /etc/pve/lxc/<CT_ID>.conf
   ```
3. Append the following device mapping and security profiles at the bottom:
   ```text
   unprivileged: 0
   lxc.apparmor.profile: unconfined
   lxc.cgroup2.devices.allow: c 226:0 rwm
   lxc.cgroup2.devices.allow: c 226:128 rwm
   lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
   lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
   ```
4. **Reboot** the LXC container. The container will now perfectly recognize the graphic device (`/dev/dri/renderD128`), allowing Ollama to use hardware GPU acceleration.

---

## ⚙️ [Step 2] Basic Packages and Dependencies Setup

Once the LXC is created, click **`Start`** and access the root account via the **`Console`** or SSH to execute the following commands in order:

```bash
# 1. Update and upgrade packages
apt update && apt upgrade -y

# 2. Install essential build/network utilities and SSH server (zstd is required by Ollama)
apt install -y curl zstd net-tools git avahi-daemon openssh-server

# 3. Allow root SSH login and password authentication
sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/g' /etc/ssh/sshd_config
systemctl restart sshd
systemctl enable ssh

# 4. Install and enable Docker Engine (for Mail-Automator)
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
```

* `avahi-daemon` is a flexible mDNS utility that automatically binds the host to `superllm.local` even if the IP address changes dynamically.

---

## 🦙 [Step 3] Install Ollama and Set External Binding (0.0.0.0)

By default, Ollama only listens on localhost (`127.0.0.1`). To communicate with other projects, you must reconfigure the environment variable to listen on **all IP ranges (`0.0.0.0`)**.

### 1. One-Click Ollama Installation
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 2. Override OLLAMA_HOST Environment Variable
Override the systemd service unit to allow external queries (from Home Assistant, Mail-Automator, etc.) directly to port `11434`.

```bash
# Create drop-in directory for systemd service
mkdir -p /etc/systemd/system/ollama.service.d

# Inject override configuration
cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_IGPU_ENABLE=1"
EOF

# Reload daemon and restart service
systemctl daemon-reload
systemctl restart ollama
```

### 3. Verify Binding Status
```bash
# Check if port 11434 is listening on *:11434 or 0.0.0.0:11434
netstat -tlnp | grep 11434
```

---

## 📥 [Step 4] Download Essential AI Models (qwen2.5:3b & moondream)

> [!NOTE]
> **About Quantization**: 
> Downloading an Ollama model without a specific tag automatically pulls a **lightweight model with 4-bit quantization (Q4_K_M)** by default. This minimizes performance degradation while drastically reducing storage (approx. 1.7~1.9GB) and CPU load, ensuring optimal speed and accuracy on low-power CPUs like the Intel N150/N95.

Run the following commands to safely pull the text and vision models:

```bash
# 1. Pull lightweight text model for Korean QA and MCP tool logic (approx. 1.9 GB)
ollama pull qwen2.5:3b

# 2. Pull multimodal vision model for QR OCR and image analysis (approx. 1.7 GB)
ollama pull moondream

# 3. Verify downloaded models
ollama list
```

> [!TIP]
> **Why qwen2.5:3b instead of 1.5b?**
> * **Intelligence Gap**: The `1.5b` model lacks the parameters needed for reliable smart home control (tool/parameter judgment) and often distorts Korean context.
> * **Hardware Capability**: Despite its 6W TDP, the Intel N150's x86 4-core Gracemont CPU is powerful enough to smoothly run the `qwen2.5:3b` model.
> * **Resource Optimization**: Unifying around the single smarter `qwen2.5:3b` model monopolizes Ollama RAM caching, providing better system responsiveness than running multiple smaller models simultaneously.

---

## 🧪 [Step 5] Final API Communication Test

Verify that Ollama's OpenAI-compatible chat completions API returns proper JSON by sending this curl command:

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:3b",
    "messages": [
      {"role": "user", "content": "Hello! Answer shortly in one word."}
    ],
    "max_tokens": 10
  }'
```

If it returns a valid JSON response like `{"choices": [{"message": {"content": "..."}}]}`, the AI integration server setup is complete!

---

## 🏎️ [Step 6] Verifying iGPU Hardware Acceleration Performance

How to check if the Intel N150 integrated graphics passed through to LXC are actively accelerating inference.

### 1. Check GPU Device Recognition in LXC
Run this command in the container shell:
```bash
ls -la /dev/dri
```
* **Expected Result**: `card0` and `renderD128` device files should be visible.

### 2. Check Ollama GPU Detection Logs
Verify if Ollama successfully acquired the graphics driver:
```bash
journalctl -u ollama --no-pager -n 50
```
* **Expected Result**: Logs should contain mentions of **`Vulkan`**, **`UHD Graphics`**, or **`initialization`** successes. `CPU-only` or `fallback` indicates an issue.

### 3. Real-time GPU Load Monitoring
Use Intel's GPU monitoring tool (`intel_gpu_top`) to observe load spikes during inference.

```bash
# 1. Install tool
apt install -y intel-gpu-tools

# 2. Run monitor
intel_gpu_top
```
* **Test**: Send a heavy prompt in another terminal:
  ```bash
  curl http://localhost:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "qwen2.5:3b",
      "messages": [{"role": "user", "content": "Explain the birth of the universe and black holes in detail."}]
    }'
  ```
* **Success**: If the **`Render/3D` engine load spikes to 50%-100%** while generating the answer, iGPU acceleration is fully active.

### 4. Inference Speed (Tokens Per Second) Comparison
Check the timing logs via `journalctl -u ollama`:
* **CPU-only mode**: Approx. **`5~8 tokens per second`**.
* **iGPU Acceleration mode**: Approx. **`12~18+ tokens per second`** (A 1.5x to 2x+ speedup).

### 5. Troubleshooting: Fallback to CPU / VRAM "0 B"
If Ollama fails to load the iGPU and falls back to CPU, execute these fixes inside the LXC container immediately:

#### Fix 1: Open Device File Permissions
The `ollama` user might lack permission to access the passed-through driver files:
```bash
chmod -R 777 /dev/dri
```

#### Fix 2: Install Missing Intel GPU & Vulkan/Mesa Drivers
The container might lack the necessary hardware acceleration runtime libraries:
```bash
apt update
apt install -y mesa-vulkan-drivers va-driver-all ocl-icd-libopencl1 intel-opencl-icd clinfo
```

#### Fix 3: Restart Ollama Service
Apply the changes by restarting the daemon:
```bash
systemctl restart ollama
```
Check `journalctl -u ollama --no-pager -n 50` again to ensure `total_vram` displays the iGPU memory capacity.
