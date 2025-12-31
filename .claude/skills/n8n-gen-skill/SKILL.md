---
name: n8n-gen
description: "Generate n8n workflows by researching official templates. This skill: (1) Searches and downloads reference workflows from n8n.io using Playwright, (2) Designs a new custom workflow based on 'n8n AI Agent' guidelines, (3) Generates the final JSON file, and (4) Creates a detailed requirement documentation."
license: Proprietary
---

# n8n Workflow Generation and Documentation

## Overview

This skill enables Claude to act as an n8n Solutions Architect. It automates the process of researching existing solutions, learning from official templates, and synthesizing a production-ready AI Agent workflow JSON. It also produces comprehensive project documentation including architecture design and flowcharts.

## Workflow

### 1. Analyze and Search Strategy

- Analyze the user's natural language request to understand the automation scenario.
- Extract 2-3 core English keywords (e.g., "Reddit monitoring", "Video generation", "WooCommerce sync") to use for searching the n8n template library.
- Prepare the search query for the template acquisition step.

### 2. Acquire Reference Templates (Playwright MCP)

Use Playwright MCP to fetch reference materials from the official repository:
1.  Navigate to `https://n8n.io/workflows/`.
2.  Handle any "Use cookies" popups if they appear.
3.  Enter the generated keywords into the search box and submit.    
4.  Iterate through the results to identify relevant workflows.
5.  For the top 5-10 most relevant workflows, perform the following actions:
    - Click into the workflow detail page.
    - Locate and click the **"Use for free"** button.
    - Wait for the popup modal.
    - Click **"Copy template to clipboard[JSON]"**.
    - *Technical Note:* Intercept the clipboard content or extract the JSON data directly from the DOM.
    - Create a local JSON file named exactly as the workflow title (sanitized for filenames).
    - Paste the code and save it to:
      ```
      n8n_references/[sanitized-workflow-name].json
      ```

### 3. Generate n8n Workflow JSON

1.  Read the specific design prompt file:
    ```
    n8n AI Agent工作流设计提示词.md
    ```
    *Note: This file contains logic for AI Agent architecture, DeepSeek v3 parameter settings, and best practices.*
    
2.  Synthesize the final workflow:
    - Combine the structural patterns learned from the downloaded reference files (Step 2).
    - Apply the logic and constraints from the design prompt (Step 3).
    - Ensure the output is a valid, importable n8n JSON format.
    - Configure specific AI Agent nodes and Loop/Merge logic as required.
3.  Save the final workflow to:
    ```
    n8n_output/[timestamp]-[project-name].json
    ```

### 4. Generate Documentation

Create a markdown file named `[timestamp]-[project-name]-requirements.md` in the `n8n_output/` directory. 
The content **must** strictly follow this structure:

#### 核心功能模块

**1. /需求分析**
> 智能需求建模 - [Describe how the AI analyzed the specific user inputs, identified triggers, data sources, and output goals for this project.]

**2. /架构设计**
> AI Agent工作流设计 - [Describe the chosen solution path based on the templates found. Detail the node configuration, DeepSeek v3 model parameters, and API integrations designed for this workflow.]

**3. /构建**
> JSON工作流生成 - [Confirm the JSON generation status. Mention key technical specs like node IDs and data flow integrity.]

**4. /流程图**
> 可视化架构展示 - [**Action**: Generate a Mermaid diagram code block here that visualizes the data flow between nodes.]

**5. /状态**
> 项目进度管理 - [Provide a status summary: e.g., "Analysis: 100%, Design: 100%, Build: 100%". Provide next steps for the user (e.g., "Import to n8n and configure credentials").]

## Technical Requirements

### Dependencies
- **Node.js & pnpm**: For managing the runtime environment.
- **Playwright**: Required for headless browser interaction to scrape `n8n.io`.
- **Clipboard/DOM Access**: To extract the JSON content from the "Copy" button action.

### Directory Structure

```

root/
├── n8n AI Agent工作流设计提示词.md  # (Required Input)
├── n8n_references/                # (Intermediate: Downloaded templates)
│   ├── template-1.json
│   └── ...
└── n8n_output/                    # (Final Output)
├── YYYYMMDD-HHMMSS-project.json
└── YYYYMMDD-HHMMSS-project-requirements.md

```

### Key Implementation Notes
- **Error Handling**: If `copy template` fails or the button is missing, skip to the next template.
- **JSON Validity**: The generated JSON must be strictly valid JSON (no markdown code blocks inside the file) to ensure direct import into n8n.
- **Reference Logic**: The AI should not blindly copy references but use them to understand *how* to connect nodes for the specific domain, then rebuild using the logic defined in the local prompt file.