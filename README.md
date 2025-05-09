
# Automated Re-Encryption of SealedSecrets

_A Secure and Scalable Implementation Plan_

---

## 📖 Table of Contents

- [Introduction](#introduction)
- [Goals & Deliverables](#goals--deliverables)
- [Architecture](#architecture)
- [Component Breakdown](#component-breakdown)
- [CLI Integration](#cli-integration)
- [Security & Scalability](#security--scalability)
- [Usage & Examples](#usage--examples)
- [Challenges & Mitigations](#challenges--mitigations)
- [Testing & Validation](#testing--validation)
- [Conclusion](#conclusion)

---

<a id="introduction"></a>

## 🧭 Introduction: Why This Feature Matters

Bitnami’s [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets) provides a secure way to manage Kubernetes secrets using asymmetric encryption, allowing safe storage of secrets in Git. However, routine **key rotation** creates operational gaps—existing SealedSecrets stay encrypted with older keys unless manually updated.

Manually decrypting and re-encrypting dozens or hundreds of secrets is not scalable. This proposal introduces an automated `kubeseal reencrypt` command, designed to address that gap securely and efficiently.

---

<a id="goals--deliverables"></a>

## 🎯 Goals & Deliverables

- Scan a cluster or namespace for all SealedSecrets
- Decrypt using historical private keys
- Re-encrypt using the latest public key
- Update the SealedSecrets in-place
- Log outcome (success/failure) per secret
- Keep sensitive materials in-memory only
- Scale gracefully for large clusters

---

<a id="architecture"></a>

## 🧱 Architecture

This feature extends the `kubeseal` CLI and relies on logic from the SealedSecrets controller.

### High-Level Workflow

1. **Discover**: List all SealedSecrets via Kubernetes API
2. **Decrypt**: Use historical private keys to unlock them
3. **Re-encrypt**: Use latest public key to seal again
4. **Update**: Push the updated SealedSecrets to the cluster

---

<a id="component-breakdown"></a>

## 🔍 Component Breakdown

### 1. ClusterScanner

```go
func ListSealedSecrets(client client.Client, namespace string) ([]v1beta1.SealedSecret, error) {
    opts := []client.ListOption{client.InNamespace(namespace)}
    var sealedSecrets v1beta1.SealedSecretList
    err := client.List(context.Background(), &sealedSecrets, opts...)
    return sealedSecrets.Items, err
}
```

_Use_: Fetch all SealedSecrets across namespaces.

---

### 2. KeyManager

```go
func LoadKeys(keySecret *v1.Secret) (map[string]*rsa.PrivateKey, *rsa.PublicKey, error) {
    // Parse private keys, identify latest public key
    return privateKeys, latestPubKey, nil
}
```

_Use_: Manage key loading and rotation logic.  
[Reference: main.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/cmd/controller/main.go)

---

### 3. Decryptor

```go
func DecryptSealedSecret(ss v1beta1.SealedSecret, privateKey *rsa.PrivateKey) (*v1.Secret, error) {
    // Use existing crypto logic to decrypt
    return decryptedSecret, nil
}
```

_Use_: Decrypt SealedSecrets using matched private key.  
[Reference: crypto.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/pkg/crypto/crypto.go)

---

### 4. ReEncryptor

```go
func ReEncryptSecret(secret v1.Secret, pubKey *rsa.PublicKey) (*v1beta1.SealedSecret, error) {
    // Seal again with latest public key
    return sealedSecret, nil
}
```

_Use_: Reseal secrets using updated key.  
[Reference: client.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/cmd/kubeseal/client.go)

---

### 5. Updater

```go
func UpdateSealedSecret(client client.Client, updated *v1beta1.SealedSecret) error {
    return client.Update(context.Background(), updated)
}
```

_Use_: Patch or replace SealedSecrets efficiently.

---

### 6. Logger (Bonus)

```go
func LogSuccess(name, ns string) {}
func LogFailure(name, ns string, err error) {}
func GenerateSummary() {}
```

_Use_: Provide real-time feedback and logs.

---

<a id="cli-integration"></a>

## 🖥 CLI Integration

Add a new command to `kubeseal`:

```diff
 func main() {
     ...
+    reencrypt := app.Command("reencrypt", "Re-encrypt all SealedSecrets with the latest public key")
+    reencryptNamespace := reencrypt.Flag("namespace", "Target namespace").Default("").String()

     switch kingpin.MustParse(app.Parse(os.Args[1:])) {
+    case reencrypt.FullCommand():
+        return runReencrypt(*reencryptNamespace)
     }
 }
```

---

<a id="security--scalability"></a>

## 🛡 Security & ⚡ Scalability

### Security

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
rules:
  - apiGroups: ["bitnami.com"]
    resources: ["sealedsecrets"]
    verbs: ["get", "list", "update"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["sealed-secrets-key"]
    verbs: ["get"]
```

- Private keys never logged or stored
- Only controller should have access

### Scalability

- Concurrency via goroutines
- Pagination for large clusters
- Support `--resume-from` flag

---

<a id="usage--examples"></a>

## 📦 Usage & Examples

### Prerequisites

- `kubeseal` v0.18.0+
- RBAC configured

### Command

```bash
kubeseal reencrypt --namespace=production --dry-run --log-file=report.json
```

### Output

```
Re-encrypting 142 SealedSecrets...
✅ Success: 140 | ❌ Failed: 2
- default/api-token (Error: Invalid private key)
Report saved to report.json
```

---

<a id="challenges--mitigations"></a>

## ⚠ Challenges & Mitigations

| Challenge              | Mitigation                          |
|------------------------|-------------------------------------|
| Private Key Exposure   | Strict RBAC & memory-only handling  |
| Partial Failures       | Log + retry only failed entries     |
| Large Clusters         | Use concurrency + batching          |
| Legacy Format          | Skip and report gracefully          |

---

<a id="testing--validation"></a>

## 🧪 Testing & Validation

- Unit tests with mock Kubernetes API
- E2E tests using `kind` or `minikube`
- GitHub Actions CI for regression

---

<a id="conclusion"></a>

## ✅ Conclusion

The `kubeseal reencrypt` feature bridges a vital gap in operational security. It automates key rotation, prevents drift, and reinforces GitOps alignment.

By integrating seamlessly with the controller logic and adhering to best practices in scale, security, and DX, this feature empowers teams to manage secrets safely—without manual toil.

---

> “Security is a process, not a product.”  
> — _Bruce Schneier_
