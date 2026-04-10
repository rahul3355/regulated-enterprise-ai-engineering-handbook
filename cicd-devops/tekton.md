# Tekton Pipelines, Tasks, and Triggers

## Overview

Tekton is a cloud-native CI/CD framework that runs pipelines as Kubernetes resources. This guide covers Tekton's core concepts: Tasks, Pipelines, Workspaces, and Triggers for banking GenAI platforms.

## Task Definition

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-and-push
  namespace: banking-ci
spec:
  params:
    - name: image
      type: string
    - name: dockerfile
      type: string
      default: Dockerfile
  workspaces:
    - name: source
  results:
    - name: image-digest
      description: Digest of the built image
  steps:
    - name: build
      image: gcr.io/kaniko-project/executor:v1.9.0
      workingDir: $(workspaces.source.path)
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(params.dockerfile)
        - --context=$(workspaces.source.path)
        - --destination=$(params.image):$(git rev-parse HEAD)
        - --digest-file=$(results.image-digest.path)
    
    - name: scan
      image: aquasec/trivy:latest
      command: ["trivy"]
      args:
        - image
        - --exit-code
        - "1"
        - --severity
        - HIGH,CRITICAL
        - $(params.image):$(git rev-parse HEAD)
```

## Pipeline Definition

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: genai-ci-pipeline
spec:
  params:
    - name: repo-url
    - name: revision
      default: main
    - name: image-name
  workspaces:
    - name: shared-workspace
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.revision)
      workspaces:
        - name: output
          workspace: shared-workspace
    
    - name: test
      taskRef:
        name: run-tests
      runAfter: [clone]
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: build
      taskRef:
        name: build-and-push
      runAfter: [clone]
      params:
        - name: image
          value: $(params.image-name)
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: deploy-staging
      taskRef:
        name: deploy-to-openshift
      runAfter: [test, build]
      params:
        - name: image
          value: $(params.image-name)
        - name: environment
          value: staging
```

## Triggers

```yaml
# Trigger pipeline on git push
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: genai-trigger-template
spec:
  params:
    - name: repo-url
    - name: revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: genai-pipeline-
      spec:
        pipelineRef:
          name: genai-ci-pipeline
        params:
          - name: repo-url
            value: $(tt.params.repo-url)
          - name: revision
            value: $(tt.params.revision)
          - name: image-name
            value: quay.io/banking/genai-api
        workspaces:
          - name: shared-workspace
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 1Gi
```

## Cross-References

- **OpenShift Pipelines**: See [openshift-pipelines.md](../kubernetes-openshift/openshift-pipelines.md) for OpenShift integration
- **CI/CD Design**: See [ci-cd-design.md](ci-cd-design.md) for pipeline architecture

## Interview Questions

1. **What is Tekton? How does it differ from Jenkins?**
2. **What are Tekton Tasks, Pipelines, and PipelineRuns?**
3. **How do Workspaces work in Tekton?**
4. **How do you trigger a Tekton Pipeline on a git push?**
5. **How do you pass results between Tekton tasks?**
6. **What are the advantages of cloud-native CI/CD?**

## Checklist: Tekton Best Practices

- [ ] Tasks are reusable and composable
- [ ] Pipelines use workspaces for shared data
- [ ] Results passed between tasks via task results
- [ ] Image scanning integrated into build tasks
- [ ] Trigger templates parameterized
- [ ] Pipeline execution monitored
- [ ] PipelineRun history retention configured
- [ ] Secrets managed via Kubernetes Secrets
