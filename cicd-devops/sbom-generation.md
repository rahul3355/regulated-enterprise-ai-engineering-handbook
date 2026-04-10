# SBOM Generation: CycloneDX and SPDX

## Overview

A Software Bill of Materials (SBOM) lists all components, libraries, and dependencies in a software artifact. SBOMs are increasingly required for compliance and supply chain security in banking.

## SBOM Formats

| Format | Maintainer | Machine Readable | Human Readable | Banking Fit |
|--------|-----------|-----------------|----------------|-------------|
| CycloneDX | OWASP | JSON, XML | Yes | Recommended |
| SPDX | Linux Foundation | JSON, RDF, Tag | Yes | Good |
| Syft | Anchore | JSON, CycloneDX | Yes | Good (tool-specific) |

## SBOM Generation in CI/CD

```yaml
# .github/workflows/sbom.yml
name: SBOM Generation
on:
  push:
    branches: [main]

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Python SBOM with CycloneDX
      - name: Generate Python SBOM
        run: |
          pip install cyclonedx-bom
          cyclonedx-py --requirements requirements.txt \
            --output sbom-cyclonedx.json \
            --output-format json
      
      # Container image SBOM with Syft
      - name: Generate Image SBOM
        uses: anchore/sbom-action@v0
        with:
          image: quay.io/banking/genai-api:${{ github.sha }}
          format: cyclonedx-json
          output-file: sbom-image.json
      
      # Merge SBOMs
      - name: Merge SBOMs
        run: |
          # Use cyclonedx-cli or custom script
          python scripts/merge_sbom.py \
            --app sbom-cyclonedx.json \
            --image sbom-image.json \
            --output sbom-final.json
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom-final.json
      
      # Store in artifact registry
      - name: Upload to S3
        run: |
          aws s3 cp sbom-final.json \
            s3://banking-sboms/genai-api/${{ github.sha }}/sbom.json
```

## SBOM Content

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "serialNumber": "urn:uuid:3e671687-395b-41f5-a30f-a58921a69b79",
  "version": 1,
  "metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "tools": [
      {
        "vendor": "CycloneDX",
        "name": "cyclonedx-bom",
        "version": "3.11.0"
      }
    ],
    "component": {
      "type": "application",
      "name": "genai-api",
      "version": "1.2.0",
      "purl": "pkg:docker/quay.io/banking/genai-api@1.2.0"
    }
  },
  "components": [
    {
      "type": "library",
      "name": "flask",
      "version": "3.0.0",
      "purl": "pkg:pypi/flask@3.0.0",
      "licenses": [{"license": {"id": "BSD-3-Clause"}}]
    },
    {
      "type": "library",
      "name": "openai",
      "version": "1.10.0",
      "purl": "pkg:pypi/openai@1.10.0",
      "licenses": [{"license": {"id": "Apache-2.0"}}]
    }
  ],
  "dependencies": [...]
}
```

## Cross-References

- **Security Scanning**: See [security-scanning.md](security-scanning.md) for vulnerability scanning
- **Container Scanning**: See [container-scanning.md](container-scanning.md) for image scanning

## Interview Questions

1. **What is an SBOM and why is it important?**
2. **Compare CycloneDX and SPDX formats. Which do you prefer?**
3. **How do you generate SBOMs in a CI/CD pipeline?**
4. **What do you do with an SBOM once you have it?**
5. **How do SBOMs help with supply chain security?**
6. **What regulations require SBOMs?**

## Checklist: SBOM Generation

- [ ] SBOM generated for every build
- [ ] Both application and container SBOMs generated
- [ ] SBOM stored with build artifacts
- [ ] SBOM linked to deployment records
- [ ] SBOM validated for completeness
- [ ] License compliance checked from SBOM
- [ ] Vulnerability scanning cross-referenced with SBOM
- [ ] SBOM retention policy defined
