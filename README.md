# n8n Workflow Generator Skill

An intelligent agent skill that automates the research, design, and generation of n8n workflows. It uses Playwright to learn from official templates and synthesizes production-ready JSON files based on best practices.

## Installation

Navigate to the skill directory:

```bash
cd .claude-plugin/n8n-gen-skill

```

Install dependencies:

```bash
pnpm install

```

Install Playwright browsers (required for scraping n8n.io):

```bash
pnpm dlx playwright install chromium

```

## Configuration

Before running, ensure the **Design Prompt** file exists in your project root. This file controls the logic for the AI Agent architecture.

File path:

```text
../n8n AI Agent工作流设计提示词.md

```

*(This file should contain the "Design Philosophy" and "Build Specifications" for the AI)*

## Usage

### Command Line

You can run the skill manually for testing purposes:

```bash
node index.js --query "Your automation requirement"

# With custom options
node index.js --query "Reddit sentiment analysis" --limit 5

```

### Options

* `--query` - The natural language description of the workflow you want (required)
* `--limit` - Number of reference templates to download (default: 5)
* `--output-dir` - Directory for final results (default: "n8n_output")
* `--help` - Show help message

### Example

```bash
node index.js \
  --query "监控大疆无人机在Reddit上的负面评论，并自动发送飞书通知" \
  --limit 8

```

## Output

The skill creates two main directories and generates the following artifacts:

### 1. References (Intermediate)

Downloaded JSON templates from n8n.io:

```text
n8n_references/
├── reddit-comment-scraper.json
├── sentiment-analysis-tool.json
└── ...

```

### 2. Final Results (Output)

The synthesized workflow and documentation:

```text
n8n_output/
├── 20251223-143000-reddit-monitor.json            # Import this to n8n
└── 20251223-143000-reddit-monitor-requirements.md # Architecture docs

```

## Integration with Claude

This skill is designed to be orchestrated by Claude Code (or compatible Agentic environments). The standard flow is:

1. **Search & Acquire**: Claude triggers the skill to search n8n.io based on user keywords.
2. **Download**: The skill uses Playwright to "copy" the JSON of the top results into `n8n_references/`.
3. **Read Context**: Claude reads the local `n8n AI Agent工作流设计提示词.md`.
4. **Synthesize**: Claude combines the downloaded patterns + design prompt to generate the final JSON.
5. **Document**: Claude generates a `.md` file explaining the architecture.

## Directory Structure

```text
n8n-gen-skill/
├── skills.md              # Skill definition for Claude
├── package.json           # Dependencies
├── index.js               # Main entry point
├── src/
│   ├── search.js          # Playwright logic for n8n.io
│   └── generate.js        # JSON synthesis logic
├── n8n_references/        # Cache for downloaded templates
├── n8n_output/            # Final generated artifacts
└── README.md              # This file

```

## Key Features

* **Context-Aware**: Doesn't hallucinate nodes; learns from actual working templates.
* **Best Practices**: Enforces "AI Agent" architecture and "Sticky Note" documentation via the design prompt.
* **Automated Docs**: Generates a requirements document with architecture diagrams for every workflow.

## Notes

* **Cookie Handling**: The Playwright script automatically attempts to close "Accept Cookies" modals on n8n.io.
* **Validation**: The generated JSON is validated against standard n8n schema structures before saving.
* **Model Defaults**: Defaults to OpenAI/DeepSeek compatible nodes unless specified otherwise in the prompt.