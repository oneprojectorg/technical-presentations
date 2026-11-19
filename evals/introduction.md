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

This is a [promptfoo](https://www.promptfoo.dev/) based llm eval tool chain designed to test its
ability to work with decision information.

- [common-eval] unscientific, need input from domain experts
- Still early, "but can run the full path"
- Requires an interfence engine (Together AI or Anthropic, local) and Homebrew
- Linux users can use [linuxbrew](https://docs.brew.sh/Homebrew-on-Linux)
  - Willing to help anyone set this up
- [PromptFoo](https://www.promptfoo.dev/) is a low code setup
- PromptFoo uses [Yaml], supports other systems, like json
  - Likes yaml because is natively supports comments. can revist

# Setup

1. Install [homebrew](https://brew.sh/)
1. Project Setup
   - Two sub projects,
     - prompt foo system for different model, comparision martix root project
     - `data` download just project
     - both should have readme.md
     - feedback welcome!
1. `git clone git@github.com:oneprojectorg/common-evals.git`

# Demo

1. glm-5.3-flash, fast and check for demo.
1. Export interfence engine api key.
1. Lets use `cambridge-pb`
1. Install
   1. `brew bundle install`
   1. `just --list`
   1. `cd data`
   1. `just download cambridge-pb`
   1. `just build-cases --per-label 3 --seed 7 cambridge-classification`
   1. `just eval-generated`

1. Run Just command for model eval
   1. `just --list`
