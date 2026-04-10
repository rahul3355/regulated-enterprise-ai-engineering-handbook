---
name: "Question"
description: Ask a question about the handbook content, usage, or contribution process
title: "[Question]: "
labels: ["question", "triage"]
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Have a question? Check the [README](../../README.md), [CONTRIBUTING.md](../../CONTRIBUTING.md), or [SUPPORT.md](../../SUPPORT.md) first — your question may already be answered.

  - type: dropdown
    id: category
    attributes:
      label: Category
      options:
        - "Usage — How do I use this handbook?"
        - "Content — Questions about specific topics covered"
        - "Contribution — How do I contribute?"
        - "Licensing — Questions about the license"
        - "Other"
    validations:
      required: true

  - type: textarea
    id: question
    attributes:
      label: Your Question
      description: Please be as specific as possible.
      placeholder: What would you like to know?
    validations:
      required: true
