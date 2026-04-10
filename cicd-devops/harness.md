# Harness for CI/CD: Deployment Strategies and Verification

## Overview

Harness is a modern CI/CD platform that automates deployments with built-in verification, rollback, and progressive delivery. This guide covers Harness for banking GenAI deployments.

## Harness Pipeline

```yaml
# Harness NG Pipeline
pipeline:
  name: GenAI API Deployment
  identifier: genai_api_deployment
  projectIdentifier: banking-genai
  orgIdentifier: banking
  tags:
    app: genai-api
    env: production
  
  stages:
    - stage:
        name: Build
        identifier: build
        type: CI
        spec:
          cloneCodebase: true
          execution:
            steps:
              - step:
                  type: Run
                  name: Build and Push
                  identifier: build_push
                  spec:
                    shell: Sh
                    command: |
                      docker build -t $IMAGE_NAME:$TAG .
                      docker push $IMAGE_NAME:$TAG
              
              - step:
                  type: Run
                  name: Run Tests
                  identifier: run_tests
                  spec:
                    shell: Sh
                    command: |
                      pip install -r requirements.txt
                      pytest --cov=src/ tests/
              
              - step:
                  type: Run
                  name: Security Scan
                  identifier: security_scan
                  spec:
                    shell: Sh
                    command: |
                      trivy image $IMAGE_NAME:$TAG --exit-code 1 --severity HIGH,CRITICAL
    
    - stage:
        name: Deploy
        identifier: deploy
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: genai-api-service
          execution:
            steps:
              - step:
                  type: CanaryDeploy
                  name: Canary Deployment
                  identifier: canary_deploy
                  spec:
                    instanceUnitType: Count
                    instances: 1
              
              - step:
                  type: Verify
                  name: Canary Verification
                  identifier: canary_verify
                  spec:
                    timeout: 10m
                    retry:
                      count: 3
                      atRuntimeMultiplier: 2
                    criteria:
                      metrics:
                        - metric: error_rate
                          condition: LESS_THAN
                          value: 1.0
                        - metric: p99_latency
                          condition: LESS_THAN
                          value: 2000
              
              - step:
                  type: RollingDeploy
                  name: Full Rollout
                  identifier: full_rollout
                  spec:
                    skipDryRun: false
              
              - step:
                  type: Verify
                  name: Post-Deployment Verification
                  identifier: post_deploy_verify
                  spec:
                    timeout: 15m
                    criteria:
                      metrics:
                        - metric: error_rate
                          condition: LESS_THAN
                          value: 0.5
            
            rollbackSteps:
              - step:
                  type: RollbackRollingDeployment
                  name: Rollback
                  identifier: rollback
```

## Verification Strategies

```yaml
# Harness verification with Prometheus
verification:
  type: Prometheus
  spec:
    address: http://prometheus.monitoring:9090
    metrics:
      - metric: http_requests_total
        query: |
          sum(rate(http_requests_total{service="genai-api",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="genai-api"}[5m]))
          * 100
        name: error_rate
        analysis:
          deviation:
            failAbove: 1.0
            warnAbove: 0.5
      - metric: http_request_duration_seconds
        query: |
          histogram_quantile(0.99, 
            rate(http_request_duration_seconds_bucket{service="genai-api"}[5m]))
          * 1000
        name: p99_latency
        analysis:
          deviation:
            failAbove: 3000
            warnAbove: 2000
      
      # GenAI-specific metrics
      - metric: genai_hallucination_rate
        query: genai_hallucination_rate{service="genai-api"}
        name: hallucination_rate
        analysis:
          deviation:
            failAbove: 5.0
            warnAbove: 2.0
```

## Cross-References

- **Progressive Delivery**: See [progressive-delivery.md](progressive-delivery.md) for deployment patterns
- **Testing in Pipelines**: See [testing-in-pipelines.md](testing-in-pipelines.md) for test automation

## Interview Questions

1. **What is Harness? How does it differ from traditional CI/CD tools?**
2. **How does Harness handle automated rollback?**
3. **What verification metrics should you check during a canary deployment?**
4. **How do you integrate GenAI-specific metrics (hallucination rate) into deployment verification?**
5. **What are Harness delegates?**
6. **How does Harness handle approval gates for production deployments?**

## Checklist: Harness CI/CD

- [ ] Pipeline defined as code (YAML)
- [ ] Build, test, and scan stages configured
- [ ] Canary deployment with verification
- [ ] Metrics-based verification criteria
- [ ] Automated rollback on verification failure
- [ ] Approval gates for production
- [ ] Rollback procedure automated
- [ ] Pipeline execution monitored
- [ ] GenAI-specific metrics in verification
