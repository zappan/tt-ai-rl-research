# Research output filenames

For every research-output Markdown file created in this repository, use this filename format:

```text
<yyyymmdd>-<descriptive-kebab-case-name>--<normalized-model-identifier>.md
```

Use the current date for `yyyymmdd`. Choose a concise, descriptive lowercase filename segment separated by single hyphens. Use the double hyphen (`--`) only as the delimiter between the descriptive name and the model identifier.

Normalize the exact model identifier to a compact lowercase filename form:

- GPT-5.5 becomes `gpt55`.
- GPT-5.6 Sol becomes `gpt56-sol`.
- GPT-5.6 Terra becomes `gpt56-terra`.

For example:

```text
20260729-ai-trace-storage-research--gpt55.md
20260729-ai-trace-storage-research--gpt56-sol.md
20260729-ai-trace-storage-research--gpt56-terra.md
```

Use the exact runtime model identity when it is available. Never infer or guess a model version or variant. If the exact model version or variant is not exposed to the agent, the agent must ask the user for it before choosing the filename. Do not use an `unknown` model suffix and do not create or rename the research-output file until the user supplies the exact model identifier.

# Research writing structure

Use an inverted-pyramid structure for executive summaries, final research summaries, decision memos, and research syntheses:

- Begin with a concise problem statement.
- Immediately state the strongest supported conclusion or preferred direction and its main reasoning.
- Follow with alternatives, tradeoffs, supporting evidence, implementation detail, and validation criteria in decreasing order of importance.
- Organize the document so material can be removed from the bottom without losing the central conclusion.
- When the research does not support a preference, lead with the strongest established findings and the unresolved decision instead of inventing a recommendation.

This structure applies to polished, decision-oriented research outputs. Raw research notes, transcripts, experiment logs, source captures, and chronological investigations may preserve their natural sequence when the evidence trail is more important than a conclusion-first presentation.
