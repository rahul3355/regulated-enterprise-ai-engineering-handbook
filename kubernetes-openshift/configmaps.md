# ConfigMaps: Configuration Management and Hot Reload

## Overview

ConfigMaps store non-sensitive configuration data as key-value pairs, separating configuration from container images. This guide covers ConfigMap patterns, hot reloading, and banking-specific configuration management.

## ConfigMap Fundamentals

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: genai-api-config
  namespace: banking-genai
data:
  # Individual keys
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_RETRIES: "3"
  EMBEDDING_MODEL: "text-embedding-3-large"
  EMBEDDING_DIMENSIONS: "1536"
  CHUNK_SIZE: "1000"
  CHUNK_OVERLAP: "200"
  VECTOR_SEARCH_TOP_K: "5"
  
  # Multi-line configuration
  config.yaml: |
    server:
      host: 0.0.0.0
      port: 8080
      workers: 4
    
    database:
      pool_size: 20
      max_overflow: 10
      pool_recycle: 1800
    
    redis:
      url: redis://redis.banking-data:6379/0
    
    genai:
      provider: openai
      model: gpt-4-turbo
      temperature: 0.1
      max_tokens: 4096
      system_prompt: |
        You are a banking assistant. Always be accurate and cite sources.
    
    rate_limiting:
      requests_per_minute: 100
      burst: 20
```

## Using ConfigMaps

```yaml
# 1. Environment variables
apiVersion: v1
kind: Pod
metadata:
  name: genai-api
spec:
  containers:
    - name: api
      image: quay.io/banking/genai-api:1.0.0
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: genai-api-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: genai-api-config
              key: LOG_LEVEL
---
# 2. Volume mount (file-based)
apiVersion: v1
kind: Pod
metadata:
  name: genai-api
spec:
  containers:
    - name: api
      image: quay.io/banking/genai-api:1.0.0
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: genai-api-config
---
# 3. All env vars from ConfigMap
apiVersion: v1
kind: Pod
metadata:
  name: genai-api
spec:
  containers:
    - name: api
      image: quay.io/banking/genai-api:1.0.0
      envFrom:
        - configMapRef:
            name: genai-api-config
```

## Hot Reload Patterns

```python
"""Hot reload configuration from ConfigMap volume mount."""
import os
import time
import json
import yaml
import logging
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

logger = logging.getLogger(__name__)

class ConfigReloader(FileSystemEventHandler):
    """Watch ConfigMap mount and reload configuration on change."""
    
    def __init__(self, config_path: str):
        self.config_path = config_path
        self.config = self._load_config()
    
    def _load_config(self) -> dict:
        """Load configuration from YAML file."""
        try:
            with open(self.config_path, 'r') as f:
                return yaml.safe_load(f)
        except Exception as e:
            logger.error(f"Failed to load config: {e}")
            return self.config
    
    def on_modified(self, event):
        """Handle ConfigMap update (K8s updates the symlink)."""
        if event.src_path == self.config_path:
            logger.info("ConfigMap updated, reloading configuration")
            new_config = self._load_config()
            self._apply_config(new_config)
            self.config = new_config
    
    def _apply_config(self, config: dict):
        """Apply new configuration."""
        # Update runtime settings
        logger.info(f"New log level: {config.get('log_level', 'info')}")
        # Apply other dynamic settings...

# Start watcher
CONFIG_PATH = "/app/config/config.yaml"
observer = Observer()
observer.schedule(ConfigReloader(CONFIG_PATH), os.path.dirname(CONFIG_PATH))
observer.start()

# Alternative: Poll-based reload (simpler)
def poll_config_reload(config_path: str, interval: int = 10):
    """Poll for ConfigMap changes."""
    import hashlib
    
    last_hash = None
    while True:
        try:
            with open(config_path, 'rb') as f:
                content = f.read()
            current_hash = hashlib.md5(content).hexdigest()
            
            if current_hash != last_hash:
                logger.info("Configuration changed, reloading")
                last_hash = current_hash
                # Reload config
                with open(config_path, 'r') as f:
                    config = yaml.safe_load(f)
                # Apply...
        except Exception as e:
            logger.error(f"Config reload error: {e}")
        
        time.sleep(interval)
```

## Cross-References

- **Secrets**: See [secrets.md](secrets.md) for sensitive configuration
- **Deployments**: See [deployments.md](deployments.md) for pod configuration

## Interview Questions

1. **How do ConfigMaps work? What are the two ways to consume them in a pod?**
2. **How does hot reload work with ConfigMap volume mounts?**
3. **What happens to a pod when a ConfigMap is updated?**
4. **What are the limitations of ConfigMaps?**
5. **How do you manage environment-specific configuration (dev vs prod)?**
6. **When would you use a ConfigMap vs environment variables vs a config file?**

## Checklist: ConfigMap Best Practices

- [ ] ConfigMaps used for non-sensitive configuration only
- [ ] Sensitive data stored in Secrets, not ConfigMaps
- [ ] ConfigMap data validated before deployment
- [ ] Hot reload implemented for runtime configuration changes
- [ ] ConfigMap naming follows convention (<app>-config)
- [ ] Version-controlled ConfigMap manifests
- [ ] Environment-specific ConfigMaps managed separately
- [ ] ConfigMap size monitored (max 1MB)
