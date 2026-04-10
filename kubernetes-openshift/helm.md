# Helm Charts: Templating and Best Practices

## Overview

Helm is the package manager for Kubernetes, enabling templated, versioned, and reusable application deployments. This guide covers Helm chart structure, templating, and banking-specific deployment patterns.

## Chart Structure

```
banking-genai-api/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── values-prod.yaml    # Production overrides
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── networkpolicy.yaml
│   └── tests/
│       └── test-connection.yaml
└── README.md
```

## Chart.yaml

```yaml
apiVersion: v2
name: banking-genai-api
description: GenAI API service for banking platform
type: application
version: 1.0.0          # Chart version
appVersion: "1.2.0"     # Application version
maintainers:
  - name: GenAI Platform Team
    email: genai-team@bank.com
dependencies:
  - name: redis
    version: "17.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

## Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "banking-genai-api.fullname" . }}
  labels:
    {{- include "banking-genai-api.labels" . | nindent 4 }}
    app.kubernetes.io/component: api
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "banking-genai-api.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.rollingUpdate.maxUnavailable }}
  template:
    metadata:
      labels:
        {{- include "banking-genai-api.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "banking-genai-api.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if .Values.readinessProbe.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path }}
              port: http
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
          {{- end }}
```

## Values File

```yaml
# values.yaml
replicaCount: 3

image:
  repository: quay.io/banking/genai-api
  pullPolicy: IfNotPresent
  tag: ""

rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

env:
  APP_ENV: "production"
  LOG_LEVEL: "info"

readinessProbe:
  enabled: true
  path: /health/ready
  initialDelaySeconds: 10
  periodSeconds: 5

redis:
  enabled: true
```

## Helm Operations

```bash
# Install chart
helm install genai-api ./banking-genai-api -n banking-genai

# Install with custom values
helm install genai-api ./banking-genai-api -n banking-genai -f values-prod.yaml

# Upgrade
helm upgrade genai-api ./banking-genai-api -n banking-genai --set image.tag=1.3.0

# Rollback
helm rollback genai-api 1 -n banking-genai  # Rollback to revision 1

# View history
helm history genai-api -n banking-genai

# Test
helm test genai-api -n banking-genai

# Package
helm package ./banking-genai-api

# Lint
helm lint ./banking-genai-api

# Template rendering (dry run)
helm template genai-api ./banking-genai-api -n banking-genai
```

## Cross-References

- **Deployments**: See [deployments.md](deployments.md) for deployment strategies
- **ConfigMaps**: See [configmaps.md](configmaps.md) for configuration

## Interview Questions

1. **What is Helm? What problems does it solve?**
2. **How do Helm templates and values work?**
3. **How do you rollback a failed Helm release?**
4. **What is the difference between chart version and appVersion?**
5. **How do you manage environment-specific values in Helm?**
6. **How do you test a Helm chart before deploying?**

## Checklist: Helm Best Practices

- [ ] Chart.yaml with proper metadata
- [ ] Default values.yaml with sensible defaults
- [ ] Environment-specific values files
- [ ] Labels follow Kubernetes conventions
- [ ] Templates use helper functions (_helpers.tpl)
- [ ] Resource limits defined in values
- [ ] Chart linted before deployment
- [ ] Chart tested with dry-run
- [ ] Helm releases monitored for health
- [ ] Chart repository configured for sharing
