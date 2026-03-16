# repoindex

A collection of GitHub Copilot agent customizations for analyzing and documenting software repositories.

## Agents

| Agent | Description |
|-------|-------------|
| `speckitext.repo-overview` | Generates a high-level project introduction, architecture overview, and getting started guide |
| `speckitext.repo-architecture` | Produces deep architectural analysis covering project structure, components, dependencies, and performance |
| `speckitext.repo-module` | Analyzes an individual module — business context, API endpoints, data models, and file classification |

## How to Use

### Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/) with the GitHub Copilot extension installed
- A workspace containing the repository you want to analyze

### Setup

1. Copy the `.github/agents/` folder from this repository into the root of the repository you want to analyze.

### Running an Agent

1. Open the target repository in VS Code.
2. Open the Copilot Chat panel (`Ctrl+Alt+I` / `Cmd+Alt+I`).
3. Switch to **Agent** mode using the mode selector in the chat input.
4. Select the agent you want to run:
   - `speckitext.repo-overview` — for a full project overview
   - `speckitext.repo-architecture` — for architectural analysis
   - `speckitext.repo-module` — for a specific module analysis
5. Type your request and send it.

**Example prompts:**

```
# Overview
Analyze this repository and generate a project overview.

# Architecture
Analyze the architecture of this repository.

# Module
Analyze the 'payments' module.
```

### Outputs

All generated documents are saved to `.github/speckit/repo_index/` inside the analyzed repository:

| Agent | Output Files |
|-------|-------------|
| `repo-overview` | `overview.md` |
| `repo-architecture` | `architecture_profile.md`, `architecture_fileindex.json` |
| `repo-module` | `<module_name>_profile.md`, `<module_name>_fileindex.json` |
