# Custom LLM Setup: RunPod & Trooper.ai

This document outlines how to configure Nunzi to work with custom LLM backends, specifically RunPod (for GLM-5 30B) and Trooper.ai (for Qwen 3 Coder 14B).

Nunzi uses [LiteLLM](https://docs.litellm.ai/) under the hood, which means it can connect to any OpenAI-compatible API endpoint.

## 1. RunPod Configuration (GLM-5 30B on RTX 4090)

Since you are running GLM-5 30B on an RTX 4090 (24GB VRAM), you must use a quantized version of the model (4-bit / GPTQ / AWQ) to fit in VRAM.

### Step 1: Deploy on RunPod
1.  Launch a RunPod instance with an **RTX 4090**.
2.  Use a template that supports **Text Generation Inference (TGI)** or **vLLM**.
3.  Expose the API on port `8000`.
4.  Ensure the endpoint is OpenAI-compatible (vLLM does this by default).

### Step 2: Configure Nunzi
You can configure this via the UI Settings or by setting environment variables in `.env` (or exporting them).

**Environment Variables:**
```bash
# The 'openai/' prefix tells LiteLLM to use the generic OpenAI client
LLM_MODEL="openai/glm-5-30b"
LLM_BASE_URL="https://YOUR-RUNPOD-ID-8000.proxy.runpod.net/v1"
LLM_API_KEY="dummy-key"  # RunPod usually uses network auth, but a non-empty key is often required by client libs
LLM_MAX_INPUT_TOKENS=8192
LLM_MAX_OUTPUT_TOKENS=4096
```

**UI Settings:**
*   **Provider:** OpenAI (Custom)
*   **Model:** `glm-5-30b`
*   **Base URL:** `https://YOUR-RUNPOD-ID-8000.proxy.runpod.net/v1`
*   **API Key:** `dummy-key`

---

## 2. Trooper.ai Configuration (Qwen 3 Coder 14B)

Trooper.ai provides an OpenAI-compatible API. You will need your API key from their dashboard.

### Step 1: Get Credentials
1.  Log in to [Trooper.ai](https://www.trooper.ai/).
2.  Generate an API Key.
3.  Note the API Base URL (usually `https://api.trooper.ai/v1` or similar).

### Step 2: Configure Nunzi
To switch to Trooper.ai, update your configuration:

**Environment Variables:**
```bash
LLM_MODEL="openai/qwen-3-coder-14b"
LLM_BASE_URL="https://api.trooper.ai/v1"
LLM_API_KEY="tr-..." # Your actual Trooper API key
```

**UI Settings:**
*   **Provider:** OpenAI (Custom)
*   **Model:** `qwen-3-coder-14b`
*   **Base URL:** `https://api.trooper.ai/v1`
*   **API Key:** `tr-...`

---

## 3. Switching Between Models

You can easily switch between these configurations using **Config Profiles** (if using the config file) or simply by changing the settings in the Nunzi UI's "LLM" tab.

### Best Practice: `config.toml`
You can define multiple profiles in your `config.toml` (if running locally):

```toml
[llm]
model = "openai/glm-5-30b"
base_url = "https://YOUR-RUNPOD-ID.proxy.runpod.net/v1"
api_key = "dummy"

[trooper]
model = "openai/qwen-3-coder-14b"
base_url = "https://api.trooper.ai/v1"
api_key = "tr-..."
```
