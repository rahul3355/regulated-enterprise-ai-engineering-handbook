---
name: "Feature Request"
description: Suggest new content, sections, or improvements
title: "[Feature]: "
labels: ["enhancement", "triage"]
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Thanks for suggesting an improvement. Please be as specific as possible so we can evaluate and prioritize your suggestion.

  - type: dropdown
    id: content-type
    attributes:
      label: Content Type
      description: What type of content are you proposing?
      options:
        - "New section or folder"
        - "New file within existing section"
        - "Additional examples or code samples"
        - "New exercise or interview question"
        - "New template"
        - "Improved existing content"
        - "Other"
    validations:
      required: true

  - type: input
    id: section
    attributes:
      label: Target Section
      description: Which section would this be added to? (Or "New section" if proposing a new folder)
      placeholder: e.g., security/, genai-platforms/, new section
    validations:
      required: true

  - type: textarea
    id: proposal
    attributes:
      label: Proposal
      description: Describe the content you want added. Include specific topics, examples, and structure.
      placeholder: What should this content cover?
    validations:
      required: true

  - type: textarea
    id: problem
    attributes:
      label: Problem This Solves
      description: What gap does this fill? Why is it needed?
      placeholder: What problem does this address?
    validations:
      required: true

  - type: textarea
    id: references
    attributes:
      label: References
      description: Any links, documents, or examples that inform this content?
    validations:
      required: false

  - type: checkboxes
    id: terms
    attributes:
      label: Confirmation
      options:
        - label: I have searched existing issues and this hasn't been proposed
          required: false
        - label: I am willing to contribute this content myself
          required: false
