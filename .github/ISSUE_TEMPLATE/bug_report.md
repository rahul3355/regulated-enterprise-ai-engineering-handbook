---
name: "Bug Report"
description: Report incorrect or broken content in the handbook
title: "[Bug]: "
labels: ["bug", "triage"]
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Thanks for taking the time to report an issue. This helps improve the handbook for everyone.

  - type: input
    id: file-path
    attributes:
      label: File Path
      description: Which file or section has the issue?
      placeholder: e.g., backend-engineering/python/fastapi.md
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Description
      description: What is wrong? What did you expect to see?
      placeholder: Describe the issue in detail...
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected Content
      description: What should the content be instead?
      placeholder: Describe the expected/corrected content...
    validations:
      required: true

  - type: dropdown
    id: severity
    attributes:
      label: Content Severity
      description: How severe is this issue?
      options:
        - "Factual Error — Incorrect technical information"
        - "Outdated — Information is no longer current"
        - "Missing Context — Important information is absent"
        - "Broken Link — Link or reference doesn't work"
        - "Typo/Formatting — Minor presentation issue"
    validations:
      required: true

  - type: textarea
    id: additional
    attributes:
      label: Additional Context
      description: Any supporting information, references, or screenshots?
    validations:
      required: false
