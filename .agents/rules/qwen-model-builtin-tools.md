# Qwen Model Built-in Tools (Harness Tools)

Reference for the built-in **Harness tools** in Qwen models offered via the QwenCloud **Token Plan**: server-side capabilities (web search, code interpreter, web scraping, two image-search tools) that a supported model invokes on its own — no tool configuration in the coding client. Applicable to **Token Plan only, not Coding Plan**. Source: [Integrate Harness tools](https://docs.qwencloud.com/token-plan/best-practices/built-in-tools.md) (verified 2026-09-05).

## Tool overview

| Tool                 | What it does                                                                                                                       |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Web search           | Retrieves information from the internet and generates answers based on search results.                                             |
| Code interpreter     | Writes and runs Python code in a sandbox environment — e.g. mathematical calculations, data analysis.                              |
| Web scraping         | Accesses a specified URL and extracts its content to give the model the information it needs.                                       |
| Reverse image search | Searches for visually similar images from an input image — finding similar products, visual content tracing.                        |
| Text-to-image search | Searches for relevant images from text descriptions — visual Q&A, image recommendation.                                            |

## Model support

Personal Edition matrix below; **Team Edition is identical** (verified 2026-09-05).

| Model         | Supported tools                                                                  |
| ------------- | -------------------------------------------------------------------------------- |
| qwen3.8-max   | All five tools                                                                   |
| qwen3.8-flash | All five tools                                                                   |
| qwen3.7-max   | Web search, code interpreter, web scraping (no image-search tools)               |
| qwen3.7-plus  | All five tools                                                                   |

## Pricing and usage

- **Billing:** per successful invocation; fees are deducted from plan Credits.
- **Activation:** none — switch the coding tool's model to one of the supported models above and ask in conversation; the model automatically invokes the matching built-in tool.

Decision points:

- Which model supports which built-in tool, or Harness-tool cost / plan applicability → this file.
- Question about the **Qwen Code CLI** (usage, configuration, features) → use the lookup rule in [qwen-code-docs.md](qwen-code-docs.md) instead — different product, different docs site.

## Refreshing this file

- Raw markdown (canonical): append `.md` to the page URL — `https://docs.qwencloud.com/token-plan/best-practices/built-in-tools.md`.
- Docs-wide index: `https://docs.qwencloud.com/llms.txt`.
- **Divergence gotcha:** the rendered HTML can diverge from the raw `.md` export (observed 2026-09-05, e.g. the `qwen3.8-flash` row and the Token-Plan-only note missing from HTML; no longer reproducing as of 2026-09-10) — when re-verifying, fetch the raw `.md` and treat it as canonical.
