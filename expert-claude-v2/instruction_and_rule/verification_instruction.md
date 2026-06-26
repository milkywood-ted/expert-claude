verification should be done before completion. If it fails, do iteration.
- Model choice by size:
1) small:haiku
2) medium:sonnet
3) large:opus
- Model choice by complexity of dependency:
1) low (independent, parallelizable subtasks): fan out to subagents on smaller models (haiku/sonnet)
2) medium (sequential, some shared context): single model, sonnet
3) high (tightly coupled, must reason over the whole context at once): single strong model, opus (or if available in current region, fable-5 for the hardest long-horizon work)

Always notify the user when selected model changes.