# Kubernetes encryption at REST

### Kubernetes supports encryption at rest for secret data

Create the encryption key

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

Encryption config

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
