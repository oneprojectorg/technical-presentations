---
title: Introduction to Common Evals
author: Ivan Willig
date: 2026-11-19
description: One line on what the talk covers.
---

# Common Evals

*techinical presentation*

[https://github.com/oneprojectorg/common-evals](https://github.com/oneprojectorg/common-evals)

This is a promptfoo based llm eval tool chain designed to test its
ability to work with decision information.

- Its unscientific, need input
- still early, "but can run the full path"
- To run requires a Together AI or Anthropic and Homebrew
- Linux users can use [linuxbrew](https://docs.brew.sh/Homebrew-on-Linux)
  - willing to help anyone set this up

# Setup

1. Install [homebrew](https://brew.sh/)
1. Project Setup
   - Two sub projects,
     - prompt foo system for different model, comparision martix
     - `data` download just project
     - both should have readme.md
     - feedback welcome!
1. `git checkout git@github.com:oneprojectorg/common-evals.git`
1. Run brew bundle in both (projects) Sorry
   1. brew bundle install
   1. cd data; brew bundle install
1. Export interfence engine api key.
1. Run Just command for model eval
   1. `just --list`

# Demo

1. glm-5.3-flash, fast and check for demo.
1. Lets use `cambridge-pb`
