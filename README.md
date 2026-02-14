# CCskills

A collection of Claude Code skills for development workflows, research, content extraction, and UI/UX design.

## Installation

Install all skills globally:

```bash
npx openskills install https://github.com/DankerMu/CCskills --global
```

Install a single skill:

```bash
npx openskills install https://github.com/DankerMu/CCskills/tree/master/<skill-name> --global
```

For example:

```bash
npx openskills install https://github.com/DankerMu/CCskills/tree/master/deep-research --global
```

## Skills

### Development Workflow

| Skill | Description |
|-------|-------------|
| [codeagent](./codeagent) | Multi-backend AI code tasks with Codex, Claude, and Gemini support |
| [do](./do) | Structured 5-phase feature development (Understand, Clarify, Design, Implement, Complete) |
| [omo](./omo) | Multi-agent orchestration for code analysis, bug investigation, and fix planning |
| [sparv](./sparv) | Minimal SPARV workflow (Specify, Plan, Act, Review, Vault) with risk detection |

### GitHub Workflow

| Skill | Description |
|-------|-------------|
| [gh-flow](./gh-flow) | End-to-end GitHub workflow orchestrator, from PRD to code merge |
| [gh-create-issue](./gh-create-issue) | Create GitHub issues from PRD or requirements with auto complexity detection |
| [gh-issue-implement](./gh-issue-implement) | Implement GitHub issues by number, create PR with test coverage |
| [gh-pr-review](./gh-pr-review) | Review and merge GitHub PRs with automated CI checks |
| [gh-release](./gh-release) | Generate release notes and create GitHub releases |

### Research & Analysis

| Skill | Description |
|-------|-------------|
| [deep-research](./deep-research) | Comprehensive research assistant with multi-source synthesis and citations |
| [search-layer](./search-layer) | Multi-source search with intent-aware scoring (Brave, Exa, Tavily, Grok) |
| [search-skill](./search-skill) | Search and recommend Claude Code skills from marketplaces |
| [github-explorer](./github-explorer) | Deep-dive analysis of GitHub projects (architecture, community, competitive landscape) |
| [repo-deep-dive-report](./repo-deep-dive-report) | Repository deep dive with structured reports, Mermaid diagrams, and scored evaluation |

### Content Extraction

| Skill | Description |
|-------|-------------|
| [content-extract](./content-extract) | URL-to-Markdown extraction with MinerU fallback |
| [mineru-extract](./mineru-extract) | MinerU API for converting URLs, PDFs, and Office docs to clean Markdown |
| [headless-web-viewer](./headless-web-viewer) | Render webpages via Playwright with JS support and screenshot capture |
| [codebase-spec-extractor](./codebase-spec-extractor) | Extract replicable engineering specifications from existing codebases |

### Design & Product

| Skill | Description |
|-------|-------------|
| [ui-ux-pro-max](./ui-ux-pro-max) | UI/UX design intelligence with 50 styles, 9 stacks, and comprehensive design data |
| [prototype-prompt-generator](./prototype-prompt-generator) | Generate structured prompts for UI/UX prototypes across multiple design systems |
| [product-requirements](./product-requirements) | Interactive requirements gathering, analysis, and PRD generation |
| [brainstorming](./brainstorming) | Turn vague ideas into validated designs through structured brainstorming |
| [doc-workflow](./doc-workflow) | DR-driven iterative document workflow with multi-round deep-research validation |

### Skill Creation

| Skill | Description |
|-------|-------------|
| [skill-install](./skill-install) | Install Claude skills from GitHub with automated security scanning |
| [skill-from-github](./skill-from-github) | Create skills by learning from high-quality GitHub projects |
| [skill-from-masters-skill](./skill-from-masters-skill) | Create skills by discovering and incorporating expert methodologies |
| [skill-from-notebook](./skill-from-notebook) | Extract methodologies from documents or examples to create skills |

## Acknowledgements

Some skills in this collection are inspired by or adapted from:

- [okwinds/miscellany](https://github.com/okwinds/miscellany)
- [cexll/myclaude](https://github.com/cexll/myclaude)

## License

MIT
