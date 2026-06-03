# Intel iGPU Vulkan 가속 설정 스킬 (Proxmox LXC & LocalAI)

이 문서는 Intel N95/N150 등 저전력 미니 PC 환경에서 내장 그래픽(iGPU UHD Graphics) 자원을 Proxmox LXC 컨테이너 내부로 패스스루하고, LocalAI에서 Vulkan 가속 백엔드를 안정적으로 구동하기 위한 기술적 노하우와 상세 가이드를 정리한 스킬 문서입니다.

---

## 1. Proxmox 호스트 ➔ LXC GPU 패스스루 설정

Intel UHD 내장 그래픽 카드를 컨테이너 내부에서 온전히 인식하기 위해 Proxmox 호스트 단에서 디바이스 마운트 및 권한 설정을 주입해야 합니다.

### 설정 파일 수정
1. Proxmox 호스트 터미널(`root@pve`)에서 대상 LXC 컨테이너 설정 파일을 엽니다.
   ```bash
   nano /etc/pve/lxc/<LXC_VMID>.conf
   ```
2. 파일 하단에 다음 렌더러 및 카드 마운트 정의와 Cgroup 장치 접근 규칙을 추가합니다.
   ```text
   unprivileged: 0
   lxc.apparmor.profile: unconfined
   lxc.cgroup2.devices.allow: c 226:0 rwm
   lxc.cgroup2.devices.allow: c 226:128 rwm
   lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
   lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
   ```
3. **PVE 웹 UI 추가 설정**: Proxmox Web GUI에서 **LXC 옵션(Options) -> Features** 메뉴 진입 후, 중첩 가상화 에러 방지를 위해 **Nesting**을 필수로 체크해 줍니다. 이후 LXC 컨테이너를 완전히 재부팅합니다.

---

## 2. LocalAI Vulkan Docker Compose 최적화 구성

iGPU Vulkan 연산을 활용하고 권한 문제를 우회하기 위해 설계된 전용 `docker-compose.yml` 템플릿입니다.

```yaml
services:
  localai:
    image: localai/localai:latest-gpu-vulkan
    container_name: local-ai
    privileged: true # 중첩 가상화 권한 오류 차단
    restart: always
    ports:
      - "8080:8080"
    environment:
      - DEBUG=true
      - MODELS_PATH=/models
      - THREADS=4
      - LOCALAI_FORCE_META_BACKEND_CAPABILITY=vulkan # 개행문자(\n) 버그 우회 설정
    volumes:
      - ./models:/models
    devices:
      - /dev/dri:/dev/dri # 호스트 GPU 패스스루 연동
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### 핵심 파라미터 해설
*   **`image: localai/localai:latest-gpu-vulkan`**: Vulkan 백엔드 라이브러리 및 드라이버가 사전에 빌드된 전용 이미지를 사용합니다.
*   **`privileged: true`**: 호스트 드라이버 디바이스 접근 시 퍼미션 디나이드(Permission Denied)를 완벽히 예방합니다.
*   **`LOCALAI_FORCE_META_BACKEND_CAPABILITY=vulkan`**: 내부 백엔드 감지 파일(`/run/localai/capability`) 시스템에서 감지 문자열 뒤에 들어가는 원치 않는 개행문자(`vulkan\n`) 오동작 버그를 방지하고 강제로 Vulkan 기능 활성화를 주입합니다.

---

## 3. llama-cpp Vulkan 백엔드 빌드 및 활성화 스킬

컨테이너가 기동된 후 내부에서 Vulkan 하드웨어 가속 연산을 전담할 백엔드를 직접 컴파일 및 인스톨해 주어야 합니다.

```bash
# 1. 컨테이너 내부로 llama-cpp Vulkan 백엔드 설치 명령 송신
docker exec -it local-ai /local-ai backends install llama-cpp

# 2. 설치 후 빌드 바이너리 및 설정 갱신을 위해 컨테이너 재시작
docker compose restart
```

---

## 4. Qwen 2.5 Instruct 모델 템플릿 튜닝

Qwen 2.5 Instruct 계열의 로컬 GGUF 모델 구동 시 한국어 답변 무한 루프나 프롬프트 제어 오류를 방지하기 위한 최적의 YAML 템플릿 양식입니다.

*   **`qwen-3b.yaml` 예시**:
```yaml
name: qwen-3b
backend: llama-cpp
context_size: 2048
threads: 4
gpu_layers: 15 # 메모리 사양에 따라 GPU 레이어 할당 조절 (iGPU 자원 활용)
mmap: true

parameters:
  model: qwen2.5-3b-instruct.gguf
  temperature: 0.7

# 무한 루프 및 프롬프트 탈주 차단 정지어(stop token) 설정
template:
  chat_message: |
    <|im_start|>{{.Role}}
    {{.Content}}<|im_end|>
  chat: |
    {{.Input}}
    <|im_start|>assistant
  suggested_stop_words:
    - "<|im_end|>"
    - "<|im_start|>"
    - "<|object_ref_start|>"
```
