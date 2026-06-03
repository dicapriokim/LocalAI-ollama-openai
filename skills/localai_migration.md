# LocalAI 마이그레이션 및 하이브리드 가속 환경 설정 스킬 (LocalAI Migration & Optimizations)

이 문서는 로컬 miniPC(예: N95, Vulkan 가속 환경 등)에서 LocalAI 서버를 구축하고, OpenAI API 호환 규격으로 서비스를 마이그레이션하며, Vision 및 텍스트 모델을 최적화하여 연동하는 스킬 가이드입니다.

---

## 1. LocalAI Vision 모델(moondream 등) GGUF 이중 모델 설정 스킬
LocalAI에서 `llama-cpp` 백엔드를 기반으로 Vision-Language 모델(VLM)을 구동하려면, 텍스트 가중치 모델 파일과 비전 프로젝터(mmproj) 파일이 모두 필요합니다.

* **YAML 설정 구조 (`/models/moondream.yaml`)**:
  ```yaml
  name: moondream
  backend: llama-cpp
  context_size: 2048
  threads: 4
  gpu_layers: 15
  mmap: true

  parameters:
    model: moondream2-text-model-f16.gguf
    mmproj: moondream2-mmproj-f16.gguf
    temperature: 0.1
  ```
* **주의 사항 및 트러블슈팅**:
  * Hugging Face 저장소 구조에 따라 모델명이 상이할 수 있으므로, 다운로드 전에 반드시 HF API (`https://huggingface.co/api/models/{username}/{repo}`)를 조회하여 실제 존재하는 GGUF 파일명과 매칭해야 404 에러를 방지할 수 있습니다.
  * 텍스트 모델과 비전 프로젝터 파일은 항상 쌍으로 동일 디렉토리에 다운로드되어야 합니다.

---

## 2. N95 등 저전력 CPU 환경의 LLM 온도(Temperature) 튜닝
N95 CPU 및 iGPU 하이브리드 환경에서 LocalAI 모델(예: Qwen 2.5 3B)을 최적화할 때의 기준 가이드입니다.

* **환각 제어 및 신뢰도 우선 ( temperature: 0.1 )**:
  * QR 코드 분석, 고정 규격 JSON 출력, 팩트 기반 요약 등 로직의 일관성이 중요한 경우 낮은 온도를 지정합니다.
* **유연한 대화 및 추론 우선 ( temperature: 0.7 )**:
  * 멀티 에이전트 오케스트레이터 협업, 에이전트 간 대화 조율, 자연스러운 답변 생성이 중요할 때는 온도를 적절히 높여 응답의 유연성을 확보합니다.

---

## 3. OpenAI API 호환 규격 소스코드 마이그레이션 스킬
기존 Ollama 통신 규격에서 OpenAI Chat Completion 규격으로 마이그레이션할 때 프록시 및 클라이언트단에서 유의할 코드 패턴입니다.

* **네트워크 예외 및 서버 에러 처리 보완 (JavaScript)**:
  ```javascript
  async function askLocalAI(prompt, model, isJson = false) {
      try {
          const res = await fetch("/api/ai-proxy", {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify({ prompt, model, isJson })
          });
          
          if (!res.ok) {
              const errData = await res.json().catch(() => ({}));
              const errMsg = errData.message || errData.error || `HTTP ${res.status}`;
              console.error("AI Proxy Error:", res.status, errData);
              showToast(`AI 오류: ${errMsg}`);
              return null;
          }

          const data = await res.json();
          
          if (data.error) {
              console.error("AI Server Error:", data.error);
              showToast(`AI 오류: ${data.error.message || data.error}`);
              return null;
          }

          let text = data.choices?.[0]?.message?.content || "";
          return text;
      } catch (e) {
          console.error("AI Error:", e);
          showToast("AI 네트워크 요청 실패");
          return null;
      }
  }
  ```
  * HTTP 상태 코드가 200이더라도 JSON 반환값 내부에 `error` 객체가 포함되어 있는 형태에 대응할 수 있도록 예외 분기 처리를 해주는 것이 안전합니다.
