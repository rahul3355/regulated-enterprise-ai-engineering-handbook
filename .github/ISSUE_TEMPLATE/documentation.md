---
name: "Documentation Issue"
description: Report problems with formatting, structure, or navigation
title: "[Docs]: "
labels: ["documentation", "triage"]
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Help us improve the handbook's documentation quality and usability.

  - type: dropdown
    id: issue-type
    attributes:
      label: Issue Type
      options:
        - "Broken link or reference"
        - "Formatting or rendering issue"
        - "Missing cross-reference"
        - "Table of contents needs updating"
        - "Unclear section structure"
        - "Accessibility issue"
        - "Other"
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Description
      description: What is the documentation problem?
      placeholder: Describe the issue and where you found it...
    validations:
      required: true

  - type: textarea
    id: suggestion
    attributes:
      label: Suggested Fix
      description: How would you fix this?
    validations:
      required: false
