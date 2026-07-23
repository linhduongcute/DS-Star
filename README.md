# DS-STAR: A Data Science Agentic Framework

DS-STAR (Data Science - Structured Thought and Action) is a Python-based agentic framework for automating data science tasks. It leverages a multi-agent system powered by large language models to analyze data, devise a plan, write and execute code, and iteratively refine the solution to answer a user's query.

This project is an implementation of the paper from Google Research: [DS-STAR: A State-of-the-Art Versatile Data Science Agent](https://research.google/blog/ds-star-a-state-of-the-art-versatile-data-science-agent/). [Paper](https://arxiv.org/pdf/2509.21825)

## Features

- **Agentic Workflow**: Implements a pipeline of specialized AI agents (Analyzer, Planner, Coder, Verifier, Router, Debugger, Finalyzer) that collaborate to solve data science problems.
- **Reproducibility**: Every step of the pipeline is saved, including prompts, generated code, execution results, and metadata. This allows for complete auditability and reproducibility of results.
- **Interactive & Resume-able**: Runs can be paused and resumed. The interactive mode allows for step-by-step execution.
- **Code Editing & Debugging**: Allows users to manually edit the generated code during a run and features an auto-debug agent to fix execution errors.
- **Configuration-driven**: Project settings, model parameters, and run configurations are managed through a `config.yaml` file.

## How it Works

The DS-STAR pipeline is composed of several phases and agents:

1.  **Analysis**: The `Analyzer` agent inspects the initial data files and generates summaries.
2.  **Iterative Planning & Execution**:
    *   The `Planner` creates an initial plan to address the user's query.
    *   The `Coder` generates Python code to execute the current step of the plan.
    *   The code is executed, and the result is captured.
    *   An automatic `Debugger` agent attempts to fix any code that fails.
    *   The `Verifier` checks if the result sufficiently answers the query.
    *   The `Router` decides what to do next: either finalize the plan or add a new step for refinement.
    *   This loop continues until the plan is deemed sufficient or the maximum number of refinement rounds is reached.
3.  **Finalization**: The `Finalyzer` agent takes the final code and results and formats them into a clean, specified output format (e.g., JSON).

All artifacts for each run are stored in the `runs/` directory, organized by `run_id`.

## Project Structure

```
/
├─── dsstar.py               # Main script containing the agent logic and CLI
├─── config.yaml             # Main configuration file
├─── prompt.yaml             # Prompts for the different AI agents
├─── pyproject.toml          # Project metadata and dependencies (uv format)
├─── uv.lock                 # Locked dependency versions for reproducibility
├─── .python-version         # Python version specification for uv
├─── data/                   # Directory for your data files
└─── runs/                   # Directory where all experiment runs and artifacts are stored
```

## Getting Started

### Prerequisites

- Python 3.11+
- An OpenRouter API key.
- [uv](https://docs.astral.sh/uv/) package manager (recommended)

### Installation

#### Using uv (Recommended)

1.  **Clone the repository:**
    ```bash
    git clone <repository-url>
    cd DS-Star
    ```

2.  **Install uv (if not already installed):**
    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

3.  **Install dependencies with uv:**
    ```bash
    uv sync
    ```

### Configuration

1.  **Set your API Key:**
    The default configuration uses DeepSeek V4 Flash through OpenRouter. Set its API key as an environment variable:
    ```bash
    export OPENROUTER_API_KEY='sk-or-v1-...'
    ```
    Alternatively, you can add it to the `config.yaml` file.

2.  **Customize `config.yaml`:**
    Create a `config.yaml` file in the root of the project and customize the settings. See the "Configuration" section below for details.

    ```yaml
    # config.yaml
    model_name: 'deepseek/deepseek-v4-flash'
    max_refinement_rounds: 5
    interactive: false
    # api_key: 'your-api-key' # Alternatively, place it here
    
    # Optional: Configure specific models for different agents
    agent_models:
      PLANNER: 'deepseek/deepseek-v4-flash'
      CODER: 'deepseek/deepseek-v4-flash'
      VERIFIER: 'deepseek/deepseek-v4-flash'
    ```

## Usage

Place your data files (e.g., `.xlsx`, `.csv`) in the `data/` directory.

### Starting a New Run

To start a new analysis, you need to provide the data files and a query.

Using uv:
```bash
uv run python dsstar.py --data-files file1.xlsx file2.xlsx --query "What is the total sales for each department?"
```

### Google Colab with KramaBench

Run the following cells in order. The second clone downloads the public
[KramaBench repository](https://github.com/mitdbg/KramaBench), including its
workloads and data.

```python
%pip install -q uv
# Replace the URL below with the GitHub repository/branch containing these changes.
!git clone https://github.com/YOUR_ACCOUNT/DS-Star.git
!git clone https://github.com/mitdbg/KramaBench.git
!cd DS-Star && uv sync
```

Set the OpenRouter key without writing it to the repository. In Colab, add a
secret named `OPENROUTER_API_KEY` (key icon in the left sidebar), then run:

```python
from google.colab import userdata
import os

os.environ["OPENROUTER_API_KEY"] = userdata.get("OPENROUTER_API_KEY")
```

Choose one KramaBench task and run DS-STAR. Change `DOMAIN` and `TASK_ID` as
needed. This cell reads the query and expands all files/globs declared by that
task's `data_sources`.

```python
import glob
import json
import subprocess
from pathlib import Path

DSSTAR_DIR = Path("/content/DS-Star")
KRAMA_DIR = Path("/content/KramaBench")
DOMAIN = "legal"
TASK_ID = "legal-hard-1"

tasks = json.loads((KRAMA_DIR / "workload" / f"{DOMAIN}.json").read_text())
task = next(item for item in tasks if item["id"] == TASK_ID)
sources = task["data_sources"]
if isinstance(sources, str):
    sources = [sources]

data_root = KRAMA_DIR / "data"
data_files = []
for source in sources:
    matches = glob.glob(str(data_root / "**" / source), recursive=True)
    data_files.extend(path for path in matches if Path(path).is_file())
data_files = sorted(set(data_files))

if not data_files:
    raise FileNotFoundError(f"No KramaBench data found for {TASK_ID}: {sources}")

command = [
    "uv", "run", "python", "dsstar.py",
    "--data-files", *data_files,
    "--query", task["query"],
]
subprocess.run(command, cwd=DSSTAR_DIR, check=True)
```

The final answer and all intermediate artifacts are written under
`/content/DS-Star/runs/<run_id>/`.

### Run a complete KramaBench domain on Kaggle

KramaBench has six domains: `archaeology`, `astronomy`, `biomedical`,
`environment`, `legal`, and `wildfire`. `run_kramabench.py` reads the domain's
JSON workload, loads every file under that domain's `data/<domain>/input`
directory, and runs the full DS-STAR pipeline once per task. File analyses are
cached once per domain, so later questions reuse them instead of calling the
Analyzer again for the same files.

Create a Kaggle Notebook, enable **Internet**, and run:

```python
%pip install -q uv
!git clone https://github.com/YOUR_ACCOUNT/DS-Star.git /kaggle/working/DS-Star
!git clone https://github.com/mitdbg/KramaBench.git /kaggle/working/KramaBench
!cd /kaggle/working/DS-Star && uv sync
```

Add `OPENROUTER_API_KEY` under **Add-ons → Secrets**, enable the secret for the
notebook, then load it without printing or saving the key:

```python
from kaggle_secrets import UserSecretsClient
import os

os.environ["OPENROUTER_API_KEY"] = (
    UserSecretsClient().get_secret("OPENROUTER_API_KEY")
)
```

Test one task before starting a paid batch:

```python
!cd /kaggle/working/DS-Star && uv run python run_kramabench.py \
  --kramabench-dir /kaggle/working/KramaBench \
  --domain legal \
  --limit 1 \
  --output-dir /kaggle/working/kramabench_output
```

Run every question in one domain (remove `--limit`):

```python
!cd /kaggle/working/DS-Star && uv run python run_kramabench.py \
  --kramabench-dir /kaggle/working/KramaBench \
  --domain legal \
  --output-dir /kaggle/working/kramabench_output
```

Run all six domains:

```python
!cd /kaggle/working/DS-Star && uv run python run_kramabench.py \
  --kramabench-dir /kaggle/working/KramaBench \
  --domain all \
  --output-dir /kaggle/working/kramabench_output
```

Useful alternatives:

```bash
# Run one exact task
uv run python run_kramabench.py --kramabench-dir /kaggle/working/KramaBench \
  --domain legal --task-id legal-hard-1

# Run 5 tasks beginning at index 10
uv run python run_kramabench.py --kramabench-dir /kaggle/working/KramaBench \
  --domain legal --start 10 --limit 5
```

Outputs are saved after every task:

- `kramabench_output/manifest.jsonl`: append-only task status for resuming.
- `kramabench_output/summary.csv`: latest status of each task.
- `kramabench_output/analysis_cache/<domain>/`: reusable analysis of every file.
- `kramabench_output/runs/<domain>_<task-id>/`: full DS-STAR artifacts.

Re-running the same command with the same output directory skips successful
tasks and retries unfinished/failed tasks. Kaggle includes files under
`/kaggle/working` when you choose **Save Version → Save & Run All**, so the
manifest, summary, final answers, generated code, and logs are retained as
notebook-version output.

### Resuming a Run

If a run was interrupted, you can resume it using its `run_id`.

```bash
uv run python dsstar.py --resume <run_id>
```

### Editing Code During a Run

You can manually edit the last generated piece of code and re-run it. This is useful for manual debugging or tweaking the agent's logic.

```bash
uv run python dsstar.py --edit-last --resume <run_id>
```
This will open the last code file in your default text editor (`nano`, `vim`, etc.). After you save and close the editor, the script will re-execute the modified code.

### Interactive Mode

To review each step before proceeding, use the interactive flag.

```bash
uv run python dsstar.py --interactive --data-files ... --query "..."
```

## UV Package Manager

This project uses `uv` for fast and reliable dependency management. Here are some useful commands:

### Common UV Commands

- **Install dependencies**: `uv sync`
- **Add a new dependency**: `uv add package-name`
- **Remove a dependency**: `uv remove package-name`
- **Update dependencies**: `uv sync --upgrade`
- **Run a command in the virtual environment**: `uv run python script.py`
- **Show installed packages**: `uv pip list`

### Benefits of UV

- **Speed**: uv is 10-100x faster than pip
- **Reliability**: Consistent dependency resolution with lock files
- **No virtual environment activation needed**: Use `uv run` to execute commands directly
- **Better dependency resolution**: Automatically resolves complex dependency conflicts

## Configuration

The following options are available in `config.yaml` and can be overridden by CLI arguments:

- `run_id` (string): The ID of a run to resume.
- `max_refinement_rounds` (int): The maximum number of times the agent will try to refine its plan.
- `api_key` (string): Your model provider API key.
- `model_name` (string): The OpenRouter model ID to use (default: `deepseek/deepseek-v4-flash`).
- `interactive` (bool): If true, waits for user input before executing each step.
- `auto_debug` (bool): If true, the `Debugger` agent will automatically try to fix failing code.
- `execution_timeout` (int): Timeout in seconds for code execution.
- `execution_timeout` (int): Timeout in seconds for code execution.
- `preserve_artifacts` (bool): If true, all step artifacts are saved to the `runs` directory.
- `agent_models` (dict): A dictionary mapping agent names (e.g., `PLANNER`, `CODER`) to specific model names. If not specified, `model_name` is used.

## Providers

DS-STAR supports multiple AI model providers. Each provider requires specific environment variables to be configured:

### OpenRouter (DeepSeek V4 Flash)

**Provider Identifier**: OpenRouter DeepSeek models prefixed with `deepseek/`

**Environment Variable**:
```bash
export OPENROUTER_API_KEY='sk-or-v1-...'
```

**Model Example**: `deepseek/deepseek-v4-flash`

### Google Gemini

**Provider Identifier**: Default provider (no prefix required)

**Environment Variable**:
```bash
export GEMINI_API_KEY='your-gemini-api-key'
```

**Model Examples**:`gemini-2.5-pro`, `gemini-2.0-flash`

### OpenAI

**Provider Identifier**: Models prefixed with `gpt` or `o1`

**Environment Variable**:
```bash
export OPENAI_API_KEY='your-openai-api-key'
```

**Model Examples**: `gpt-4`, `gpt-4-turbo`, `o1`

### Ollama

**Provider Identifier**: Models prefixed with `ollama/`

**Environment Variables**:
```bash
export OLLAMA_API_KEY='your-ollama-api-key'  # Optional
export OLLAMA_HOST='http://localhost:11434'  # Optional, defaults to http://localhost:11434
```

**Model Examples**: `ollama/llama3`, `ollama/qwen3-coder`
## Contributing

Contributions are welcome! Please feel free to submit a pull request or open an issue for any bugs or feature requests.

```
