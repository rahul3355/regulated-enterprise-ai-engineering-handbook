---
name: "Security Issue"
description: Report a security concern in code examples or guidance
title: "[Security]: "
labels: ["security", "triage"]
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Thank you for reporting a security concern. If this is a **sensitive or critical** finding, please also email the maintainers directly rather than filing a public issue.

        For general security questions, see [SECURITY.md](../../SECURITY.md).

  - type: input
    id: file-path
    attributes:
      label: Affected File(s)
      description: Which file(s) contain the security issue?
      placeholder: e.g., security/api-security.md, skills/secure-api-design.md
    validations:
      required: true

  - type: dropdown
    id: vulnerability-type
    attributes:
      label: Vulnerability Type
      options:
        - "Insecure code example in documentation"
        - "Missing security control in guidance"
        - "Outdated security recommendation"
        - "Exposed secret in code sample"
        - "Other"
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Description
      description: Describe the security concern in detail. Include specific code snippets or guidance that is problematic.
      placeholder: What is the security issue?
    validations:
      required: true

  - type: textarea
    id: impact
    attributes:
      label: Potential Impact
      description: What could happen if someone follows this guidance as-is?
      placeholder: Describe the potential impact...
    validations:
      required: true

  - type: textarea
    id: remediation
    attributes:
      label: Suggested Remediation
      description: How would you fix this? Include corrected code or guidance if possible.
    validations:
      required: false

  - type: textarea
    id: references
    attributes:
      label: References
      description: Links to CVEs, advisories, or documentation supporting this finding.
    validations:
      required: false
