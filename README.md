# OMNI

OMNI is a Windows-first, terminal-based coding agent designed to run against a **local GGUF model** through `llama.cpp`'s `llama-server`. It can inspect a chosen project, search and read text files, prepare and apply precise edits, run approved development commands, report actual test results, and keep persistent conversations on the same machine.

OMNI is a real application, not a chatbot mockup: inference requests go to the configured local server, and tool output comes from application code executing actual filesystem/process operations. Accuracy still depends on the selected model. Review edits and command approvals carefully.

## Quick start on Windows 10/11

1. Extract this ZIP to a normal folder.
2. Open PowerShell in the extracted folder and run `.\install.ps1` after reviewing the local script. The script checks for Python 3.11+, creates a per-user virtual environment, copies the application to `%LOCALAPPDATA%\Programs\OMNI`, creates an `OMNI` launcher and adds it to your **user** PATH. No administrator rights are requested by OMNI.
3. Open a **new PowerShell** window. Run `OMNI` and follow the setup wizard.
4. Choose one of the catalog models, register your own `.gguf` file, or skip model setup. Catalog downloads happen only after a confirmation prompt.
5. Install or select `llama-server`. On Windows the wizard can offer `winget install llama.cpp`; alternatively set `runtime_path` in `%APPDATA%\OMNI\config.json`.
6. Open PowerShell in your code project and run `OMNI`.

PowerShell may block local scripts because of execution policy. In that case, start this local installer with `powershell -ExecutionPolicy Bypass -File .\install.ps1` after you have inspected it. This setting applies to that installer process invocation and does not change the machine-wide policy.

### Manual/developer run

Requires Python 3.11+ and an installed `llama-server` binary.

```powershell
$env:PYTHONPATH = "$PWD\src"
python -m omni --version
python -m omni --setup
python -m omni doctor
python -m omni
```

Standard-library tests need no extra packages:

```powershell
$env:PYTHONPATH = "$PWD\src"
python -m unittest discover -s tests -v
```

## Model options

| Setup tier | Catalog ID | Upstream repository | Q4_K_M files / approximate size |
|---|---|---|---|
| Around 8 GB VRAM | `qwen2.5-coder-7b-q4km` | [Qwen2.5-Coder-7B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF) | Two shards, about 4.68 GiB |
| Around 12–16 GB VRAM | `qwen3-14b-q4km` | [Qwen3-14B-GGUF](https://huggingface.co/Qwen/Qwen3-14B-GGUF) | About 9 GiB (repository listing rounds to about 9 GB) |
| Around 32 GB VRAM or more | `qwen3.6-27b-q4km` | [Official base model](https://huggingface.co/Qwen/Qwen3.6-27B) · [GGUF used by OMNI](https://huggingface.co/ggml-org/Qwen3.6-27B-GGUF) | About 19.1 GiB (repository listing) |

The Q4_K_M catalog file names, upstream repositories, approximate sizes and model cards were checked on **17 October 2026**. This is a best-effort snapshot; upstream listings can change. OMNI queries current Hugging Face repository metadata before it downloads, checks that the requested files exist, resumes partial files when the server supports HTTP range requests, and verifies file sizes / SHA-256 where trustworthy metadata is available. An incomplete file is not marked ready.

Quantized file size is not the same as runtime memory use. GPU layers, context length, KV cache, compute buffers, model architecture, driver/runtime build and other GPU processes affect what fits. The setup wizard reports hardware estimates but does not guarantee a given model fits in VRAM. CPU/GPU offloading may work but can be slower. The 27B model is a very large download; check free disk and RAM first.

**Use your own model:** copy or keep a model file wherever you prefer and run:

```powershell
OMNI add-model "D:\Models\my-coder.gguf"
```

This registers the model by absolute path; OMNI does not copy, move or delete that file. Current inference backend requires GGUF supported by the installed `llama.cpp` version.

## Everyday commands

- `OMNI` — launch the interactive terminal UI from the current project directory.
- `OMNI --setup` — reopen setup.
- `OMNI models` — show model catalog and local `.gguf` files.
- `OMNI install-model qwen2.5-coder-7b-q4km` — explicitly install a catalog model.
- `OMNI add-model D:\Models\coder.gguf` — register an existing GGUF.
- `OMNI doctor` — diagnose hardware, configuration and runtime.
- `OMNI smoke-test` — launch the configured server and issue a real local inference request.
- `OMNI --version` — print the version.

Type `/help` inside OMNI for session commands. Key commands include `/status`, `/models`, `/model`, `/install-model`, `/remove-model`, `/setup`, `/plan`, `/permissions`, `/diff`, `/undo`, `/compact`, `/context`, `/cost`, `/doctor`, `/history`, `/resume`, `/new`, `/init`, and `/quit`.

## Permission modes

- **PLAN** (default): read/search/inspect only. Writes and process commands are denied. Read-only Git status/diff inspection is allowed.
- **ACCEPT-EDITS**: prompts for approval before file changes; process commands always need user confirmation.
- **AUTO**: routine file creations/edits are allowed by application policy, with diffs/backups; deletion, all process commands, and sensitive-file access still require explicit approval.

The model cannot modify its own permission mode. Commands run without a shell, from the chosen project root, through a limited executable allowlist. Destructive Git commands, installs, remote transfers, shell operators, privileged commands and several risky patterns are blocked. This allowlist is a safety layer, not a sandbox for malicious code. A user-approved test suite or build script can still execute arbitrary project code; approve commands only when you trust the repository.

## Local data and privacy

Typical Windows locations:

- Program and launcher: `%LOCALAPPDATA%\Programs\OMNI`
- Downloaded models: `%LOCALAPPDATA%\OMNI\models`
- Runtime logs: `%LOCALAPPDATA%\OMNI\logs`
- SQLite conversations and data: `%LOCALAPPDATA%\OMNI\data`
- Recoverable file backups: `%LOCALAPPDATA%\OMNI\backups`
- Configuration: `%APPDATA%\OMNI\config.json`

Normal model inference is directed to a loopback address (`127.0.0.1` by default). OMNI does not include telemetry and does not intentionally upload source code, prompts or conversations. Model download commands contact Hugging Face when explicitly requested. The runtime and dependencies remain software that must be trusted. This application currently does not implement online documentation research.

The agent ignores common generated/dependency directories and `.gitignore` patterns, skips `.vscode` by default, avoids binary files and enforces a file-size limit. Secret-like files (for example `.env`, private keys and credential filenames) are excluded from text search and blocked from reading unless the user approves a warning. Keep credentials out of prompts and logs where possible.

## Configuration

The config is JSON and is validated at startup. Common keys include `default_model`, `model_dir`, `runtime_path`, `api_host`, `api_port`, `context_length`, `max_output_tokens`, `gpu_layers`, `cpu_threads`, `batch_size`, `permission_mode`, `command_timeout_seconds`, `max_agent_iterations`, `max_tool_output_chars`, `max_file_bytes`, `ignored_paths`, `database_path`, `logging_level`, and retry/timeout settings.

- `api_host` accepts loopback values only (`127.0.0.1`, `localhost`, `::1`).
- The default endpoint is `http://127.0.0.1:8080`.
- Change `context_length` downward if the runtime cannot allocate sufficient memory.
- Reduce `gpu_layers` for partial GPU offload (runtime support/hardware permitting).
- `cpu_threads: 0` is supported in config, and setup selects a conservative value for a newly configured system.

Invalid or unknown keys are reported instead of being silently accepted. OMNI's approximate token count uses a character heuristic; it is not an exact tokenizer result.

## Troubleshooting

**`llama-server was not found`:** install a compatible `llama.cpp` build or set `runtime_path` to the full path of `llama-server.exe`. Restart the terminal after package-manager installation if needed.

**Model won't load / out of memory:** reduce `context_length`, set fewer GPU layers, close other GPU-heavy apps, check available system RAM, or choose a smaller quantization/model. The server log is `%LOCALAPPDATA%\OMNI\logs\llama-server.log`.

**Port conflict:** change `api_port` (1024–65535) in config. OMNI will not terminate a process that it did not start.

**Model download failed:** rerun `OMNI install-model <catalog-id>`; compatible `.part` files may resume. Verify free disk space and connectivity to Hugging Face. No network download occurs just from chatting.

**Agent tool syntax fails:** the current agent uses a strict structured text protocol because GGUF models vary in tool calling. After repeated malformed actions OMNI stops instead of executing ambiguous text. Try a supported instruction-tuned coding model.

**Command denied:** type `/help` and read `/permissions`. Approval prompts are enforced by code; the model cannot override them.

## Uninstall

Run `uninstall.ps1` from the extracted project folder. It asks separately whether to remove application files, local data/models/backups, and configuration. Answering no preserves that category. Existing PowerShell sessions may need to restart after PATH cleanup.

