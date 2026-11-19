---
title: Introduction to Common Evals
author: Ivan Willig
date: 2026-11-19
description: One line on what the talk covers.
---

# Common Evals

*techinical presentation*, engineers and research folks

- [https://github.com/oneprojectorg/common-evals](https://github.com/oneprojectorg/common-evals)

- [https://oneprojectorg.github.io/technical-presentations/evals/introduction.html](https://oneprojectorg.github.io/technical-presentations/evals/introduction.html)

```
This is a promptfoo based llm eval tool chain designed to test an LLM's model
ability to work with decision information.
```

- Common-eval is unscientific, need input from domain experts
- Still early, "but can run the full path"
- Requires an interfence engine (Together AI or Anthropic, local) and Homebrew
- Linux users can use [linuxbrew](https://docs.brew.sh/Homebrew-on-Linux)
  - Willing to help anyone set this up
- [PromptFoo](https://www.promptfoo.dev/) is a low code setup
- [Yaml](https://yaml.org/)
  - promptfoo also supports JSON
  - Yaml because is natively supports comments. Can revist

# Setup

1. Install [homebrew](https://brew.sh/)
1. Project Setup
   - Two sub projects,
     - [root project](https://github.com/oneprojectorg/common-evals): prompt foo system for different model, comparision martix
     - [data](https://github.com/oneprojectorg/common-evals/tree/main/data): data catalog, and document on how to process public data for evaluation.
     - both should have readme.md
     - feedback welcome!
1. `git clone git@github.com:oneprojectorg/common-evals.git`

# Demo

1. glm-5.3-flash, fast and cheap for demo.
1. Export interfence engine api key.
1. Use `cambridge-pb`
1. Install
   1. `brew bundle install`
   1. `just --list`
   1. `cd data`
   1. `just download cambridge-pb`
   1. `just build-cases --per-label 3 --seed 7 cambridge-classification`
   1. `just eval-generated`
   1. `just grader=huggingface:chat:zai-org/GLM-5.3 eval-generated hf-glm-5.3 hf-glm-5.3-flash hf-qwen-27b`
   1. `just --list`
