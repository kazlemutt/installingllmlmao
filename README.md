# Local AI Coding Setup: Ollama + Pi (`pi.dev`) in VS Code

Run **`gemma-4-26B-A4B-it-ultra-uncensored-heretic` (`Q4_K_M`)** locally inside VS Code using a hybrid **6GB GPU VRAM + 24GB System RAM** split. Because this model uses a Mixture-of-Experts (MoE) architecture (`26B total · 4B active`), only 4B parameters are activated per token—keeping inference fast even when split across system RAM.

---

## Hardware & Memory Profile

| Component | Target Spec | Allocation |
| :--- | :--- | :--- |
| **GPU VRAM** | NVIDIA RTX 3050 (6 GB) | ~4.8 GB (`12` offloaded layers) |
| **System RAM** | 24 GB DDR5 | ~15.2 GB (remaining layers + 32K KV cache) |
| **Model Download** | `Q4_K_M` GGUF | ~16.8 GB on disk |
| **Context Window** | `32,768` tokens | Configured with `8,192` token auto-compaction reserve |

> **Tip:** Close heavy browser tabs and background apps before launching so Windows and VS Code have plenty of RAM headroom.

---

## 1. Download the Model in Ollama

Pull the `Q4_K_M` quantization from Hugging Face:

```bash
ollama pull hf.co/mradermacher/gemma-4-26B-A4B-it-ultra-uncensored-heretic-GGUF:Q4_K_M
```

Verify the download:

```bash
ollama list
```

---

## 2. Create a Custom `Modelfile` for Hybrid GPU/RAM Splitting

Create a folder (for example, `C:\ollama-models` on Windows or `~/ollama-models` on macOS/Linux) and save a file named `Modelfile` (**with no `.txt` extension**) containing:

```dockerfile
FROM hf.co/mradermacher/gemma-4-26B-A4B-it-ultra-uncensored-heretic-GGUF:Q4_K_M
PARAMETER num_gpu 12
PARAMETER num_ctx 32768
```

Build the custom split profile:

```bash
cd C:\ollama-models
ollama create gemma4-26b-split -f ./Modelfile
```

### Verify the Split

1. Start a test prompt:
   ```bash
   ollama run gemma4-26b-split "hello"
   ```
2. In a second terminal, check the GPU/CPU split and VRAM usage:
   ```bash
   ollama ps
   nvidia-smi
   ```
   Confirm `ollama ps` shows a `GPU/CPU` split with `CONTEXT` at `32768`, and `nvidia-smi` stays below `5600MiB / 6144MiB`. Type `/bye` in the first terminal to exit.

---

## 3. Install Pi (`pi.dev`) and Link Ollama

Run Ollama's built-in setup command to install the Pi coding agent and register your local Ollama instance:

```bash
ollama launch pi --config
```

*(Alternative manual install via npm: `npm install -g @earendil-works/pi-coding-agent`)*

Verify the installation:

```bash
pi --version
```

---

## 4. Configure Pi's Context Window & Compaction

### Register the Split Model (`models.json`)

Open or create `~/.pi/agent/models.json` (`C:\Users\<YourUsername>\.pi\agent\models.json` on Windows):

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "gemma4-26b-split",
          "name": "Gemma 4 26B-A4B Heretic (Q4 Split)",
          "contextWindow": 32768,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

### Align Auto-Compaction (`settings.json`)

Open or create `~/.pi/agent/settings.json` (`C:\Users\<YourUsername>\.pi\agent\settings.json` on Windows) so Pi compacts history cleanly within your 32K window:

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 8192,
    "keepRecentTokens": 12000
  }
}
```

---

## 5. Connect VS Code & Launch Pi

1. Open your project folder in **VS Code**.
2. Press `Ctrl + P` (`Cmd + P` on macOS), paste the command below, and press `Enter` to install the VS Code bridge extension:
   ```text
   ext install pi0.pi-vscode
   ```
3. Reload VS Code (`Ctrl + Shift + P` → **Developer: Reload Window**).
4. Open VS Code's integrated terminal (`` Ctrl + ` ``) and launch Pi with your split model:
   ```bash
   ollama launch pi --model gemma4-26b-split
   ```

---

## Useful Commands & Optional Tuning

### Local Session Management
Pi automatically saves all conversations locally in `~/.pi/agent/sessions/` (`C:\Users\<YourUsername>\.pi\agent\sessions\` on Windows):

| Command | Action |
| :--- | :--- |
| `pi --continue` | Resume your most recent session in the current folder |
| `pi --resume` (or `/resume`) | Browse and search all saved sessions |
| `/session` | View current session file path, message count, and token usage |
| `/name` | Give the active session a custom label |
| `/export` | Export the conversation to standalone HTML or JSONL |
| `pi --no-session` | Start a temporary session without saving to disk |

### Optional: Cut KV Cache RAM Usage in Half
To reduce 32K context memory overhead by ~1.7 GB with minimal quality impact, set these system environment variables and restart Ollama:

```powershell
# PowerShell (Windows)
[System.Environment]::SetEnvironmentVariable("OLLAMA_FLASH_ATTENTION", "1", "User")
[System.Environment]::SetEnvironmentVariable("OLLAMA_KV_CACHE_TYPE", "q8_0", "User")
```
