---
name: ollama-check
description: 'Assess how much of a repository can run with local Ollama models. Use when testing Ollama support, local model compatibility, OpenAI-compatible APIs, gemma, qwen, gpt-oss, embeddings, RAG scripts, devcontainer readiness, or README claims about local models.'
argument-hint: 'Optional: model names and entry points to test, such as gemma4:e4b qwen3.5:4b scripts/*.py'
---

# Ollama Compatibility Assessment

Use this skill to determine how much of a repo can run with local Ollama models, and to gather evidence for docs, devcontainers, CI notes, or PRs.

The goal is not to prove every model works perfectly. The goal is to separate:

- entry points that run as-is with local Ollama chat models
- entry points that need model features such as tool calling, JSON/structured output, vision, embeddings, or reasoning
- entry points that fail because they depend on cloud-only services, hard-coded model names, missing dependencies, or interactive loops
- documentation and setup changes that are safe to recommend

## Good Default Models

Start with small local models that are already pulled, or use these tested families if available:

- `gemma4:e4b` or another small Gemma chat model
- `qwen3.5:4b` or another small Qwen chat model
- `gpt-oss:20b` for reasoning-specific checks, if available
- `nomic-embed-text` or `embeddinggemma` for embedding checks, if available

Prefer a primary/fallback model pair instead of testing every entry point against every model. For example:

- primary: `qwen3.5:4b`
- fallback: `gemma4:e4b`

For docs and devcontainer defaults, choose the smallest model that passes representative smoke tests reliably in that repo.

## Before You Start

1. Confirm repo and branch context.

   ```shell
   git remote -v
   git branch --show-current
   git status --short
   ```

2. Confirm the intended upstream/base branch before doing PR-oriented work.

   ```shell
   git fetch --no-tags origin
   git log --oneline --decorate -5
   ```

3. Check whether the repo uses Python, JavaScript, .NET, or another runtime.

   ```shell
   ls
   find . -maxdepth 2 -type f \( -name 'pyproject.toml' -o -name 'requirements*.txt' -o -name 'package.json' -o -name '*.csproj' \) | sort
   ```

4. Use the repo's own setup commands when they exist. Do not invent a new package manager if the repo already has one.

## Environment Checks

Check Ollama availability and pulled models:

```shell
command -v ollama || true
ollama list || true
curl -sS http://localhost:11434/api/tags || true
```

If a required model is missing, pull it explicitly:

```shell
ollama pull gemma4:e4b
ollama pull qwen3.5:4b
```

Check the language runtime for the repo. Python example:

```shell
python3 --version
python3 -m pip list | grep -E 'openai|python-dotenv|pydantic|langchain|llama-index|sentence-transformers|tiktoken' || true
```

JavaScript/TypeScript example:

```shell
node --version
npm --version
npm ls openai @langchain/openai langchain 2>/dev/null || true
```

## Identify Ollama Configuration Knobs

Search for provider/model/environment configuration:

```shell
rg -n 'OLLAMA|OPENAI|AZURE_OPENAI|API_HOST|BASE_URL|base_url|model|embeddings|tools|response_format|json_schema|stream' .
```

Record:

- how the repo selects a provider
- how the repo sets the model name
- how the repo sets the OpenAI-compatible base URL
- whether `.env` overrides shell environment variables
- whether scripts hard-code model names for chat, embeddings, or reasoning

Common OpenAI-compatible Ollama settings:

```env
API_HOST=ollama
OLLAMA_ENDPOINT=http://localhost:11434/v1
OLLAMA_MODEL=gemma4:e4b
OPENAI_BASE_URL=http://localhost:11434/v1
OPENAI_API_KEY=nokeyneeded
OPENAI_MODEL=gemma4:e4b
```

Use the variable names the repo already supports.

## Force Entry Points To Use Ollama

If the repo supports an Ollama-specific provider, use it:

```shell
API_HOST=ollama \
OLLAMA_ENDPOINT=http://localhost:11434/v1 \
OLLAMA_MODEL=gemma4:e4b \
python3 script.py
```

If the repo only supports generic OpenAI-compatible configuration, use:

```shell
OPENAI_BASE_URL=http://localhost:11434/v1 \
OPENAI_API_KEY=nokeyneeded \
OPENAI_MODEL=gemma4:e4b \
python3 script.py
```

If `.env` files override shell environment variables, disable dotenv if the library supports it or run with a temporary test env file. Python `python-dotenv` example:

```shell
PYTHON_DOTENV_DISABLED=true \
API_HOST=ollama \
OLLAMA_ENDPOINT=http://localhost:11434/v1 \
OLLAMA_MODEL=gemma4:e4b \
python3 script.py
```

## Choose What To Test

Group entry points by capability instead of running everything as one giant sweep.

### Basic Chat And Streaming

Test entry points that make simple chat/responses calls and streaming calls.

Pass criteria:

- exits 0
- prints a response
- streaming event parsing does not fail

### Interactive Loops

For scripts that call `input()` forever, stub one input and exit on the second prompt.

Python pattern:

```python
import builtins
import runpy
import sys

answers = iter(["Hello"])

def fake_input(prompt=""):
    print(prompt, end="", flush=True)
    try:
        return next(answers)
    except StopIteration:
        raise SystemExit(0)

builtins.input = fake_input
runpy.run_path(sys.argv[1], run_name="__main__")
```

### Tool Calling

Test whether the model returns tool calls and valid JSON arguments.

Pass criteria:

- exits 0
- returns expected tool call structures
- JSON arguments parse successfully
- any follow-up round trip works

If tool calling fails for one local model, try the fallback before declaring Ollama unsupported.

### Structured Output Or JSON Mode

Test Pydantic schemas, JSON schemas, `response_format`, and JSON-mode examples.

Pass criteria:

- exits 0
- validates against the schema
- no parser exceptions

### RAG Without Embeddings

Test retrieval flows that use CSV, keyword search, checked-in chunks, or plain text context.

Pass criteria:

- retrieval step works
- local chat model answers from supplied context

### Embedding-Based RAG

Check whether embedding model names are configurable.

Hard-coded cloud embedding model names, such as `text-embedding-3-small`, often fail against local Ollama unless the code maps them to a pulled local embedding model.

Run a direct embedding check:

```shell
python3 - <<'PY'
import openai

client = openai.OpenAI(base_url="http://localhost:11434/v1", api_key="nokeyneeded")

for model in ["text-embedding-3-small", "nomic-embed-text", "embeddinggemma"]:
    try:
        result = client.embeddings.create(model=model, input="hello world")
        print(f"EMBED {model} PASS dims={len(result.data[0].embedding)}")
    except Exception as error:
        print(f"EMBED {model} FAIL {type(error).__name__}: {error}")
PY
```

Expected local-only result:

- cloud embedding model names usually fail
- pulled local embedding models can pass

Document this limitation unless you also update the repo to configure embedding model names.

### Reasoning

Reasoning-specific examples may need a reasoning model and may expose provider-specific response fields.

Test those separately with a reasoning-capable local model, such as `gpt-oss`, if available.

## Minimal Python Batch Harness

Use this pattern for Python repos. Replace `scripts` with the repo's entry points and tune `models`.

```python
from __future__ import annotations

import os
import pathlib
import subprocess
import sys
import time

repo = pathlib.Path.cwd()
models = ["qwen3.5:4b", "gemma4:e4b"]
scripts = [
    "chat.py",
    "chat_stream.py",
]

runner = r'''
import builtins
import runpy
import sys

answers = iter(["Hello"])

def fake_input(prompt=""):
    print(prompt, end="", flush=True)
    try:
        return next(answers)
    except StopIteration:
        raise SystemExit(0)

builtins.input = fake_input
runpy.run_path(sys.argv[1], run_name="__main__")
'''

base_env = os.environ.copy()
base_env.update({
    "API_HOST": "ollama",
    "OLLAMA_ENDPOINT": "http://localhost:11434/v1",
    "OPENAI_BASE_URL": "http://localhost:11434/v1",
    "OPENAI_API_KEY": "nokeyneeded",
    "PYTHON_DOTENV_DISABLED": "true",
    "TOKENIZERS_PARALLELISM": "false",
})

for script in scripts:
    passed = False
    attempts = []
    for model in models:
        env = base_env.copy()
        env["OLLAMA_MODEL"] = model
        env["OPENAI_MODEL"] = model
        start = time.monotonic()
        try:
            proc = subprocess.run(
                [sys.executable, "-c", runner, str(repo / script)],
                cwd=repo,
                env=env,
                text=True,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                timeout=120,
            )
            elapsed = time.monotonic() - start
            output = (proc.stderr + "\n" + proc.stdout).strip().replace("\n", " / ")
            attempts.append((model, proc.returncode, elapsed, output[-500:]))
            if proc.returncode == 0:
                passed = True
                print(f"RESULT {script} PASS model={model} seconds={elapsed:.1f}", flush=True)
                break
        except subprocess.TimeoutExpired as exc:
            elapsed = time.monotonic() - start
            stdout = exc.stdout.decode(errors="replace") if isinstance(exc.stdout, bytes) else (exc.stdout or "")
            stderr = exc.stderr.decode(errors="replace") if isinstance(exc.stderr, bytes) else (exc.stderr or "")
            output = (stderr + "\n" + stdout).strip().replace("\n", " / ") or "Timed out"
            attempts.append((model, "timeout", elapsed, output[-500:]))
    if not passed:
        print(f"RESULT {script} FAIL attempts={attempts}", flush=True)
```

## How To Interpret Results

Use exit code as the first pass/fail signal, then read the output excerpt to classify the failure.

Common failure classes:

- model missing: pull the model or use a pulled model
- timeout: retry with fallback model and shorter script group
- unsupported endpoint parameter: document as provider incompatibility or add provider-specific handling
- hard-coded embedding model: document as a local-only Ollama limitation or make embedding model configurable
- missing dependency: add it to the appropriate requirements file only if it is genuinely required
- code bug: fix only if it is in scope and still exists on the current upstream branch
- cloud-only service: document as not supported by local Ollama

## Documentation Guidance

Use cautious wording:

```text
Most samples work with local Ollama chat models. Representative chat, streaming, tool calling, structured output, and non-embedding RAG flows were tested with <model names>.
```

Call out known limits explicitly:

```text
Embedding-based flows may still require cloud embeddings or a code update to configure local Ollama embedding models.
```

For endpoint guidance:

```text
Use http://localhost:11434/v1 when Ollama and the app run in the same environment. If the app runs in a container and Ollama runs on the host machine, use http://host.docker.internal:11434/v1 instead.
```

## Devcontainer Guidance

If adding an Ollama devcontainer, prefer:

- a small default model that passed smoke tests
- an `.env` sample copied on create
- clear host memory requirements
- a note that local models consume significant disk and memory

Example feature configuration:

```json
"features": {
    "ghcr.io/prulloac/devcontainer-features/ollama:1": {
        "pull": "gemma4:e4b"
    }
}
```

## Validation Before PR

Run the repo's normal validation first. Examples:

```shell
git diff --check
python3 -m py_compile path/to/changed_script.py
npm test
```

Then smoke test a representative Ollama entry point:

```shell
API_HOST=ollama \
OLLAMA_ENDPOINT=http://localhost:11434/v1 \
OLLAMA_MODEL=gemma4:e4b \
python3 chat.py
```

If testing an interactive script, use the `input()` stub pattern above so the script exits cleanly.

## output

Use the collected evidence to write documentation, devcontainer notes, or PR comments. Focus on actionable recommendations and clear explanations of limitations. 

Outline what changes need to be made in the repo to support local Ollama models, and what limitations may remain even after those changes.