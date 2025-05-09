
# Automated Re-Encryption of SealedSecrets  
*A Secure and Scalable Implementation Plan*

---



## 📖 Table of Contents  
- [Introduction](#introduction)  
- [Architecture](#architecture)  
- [Component Deep Dive](#component-deep-dive)  
- [Security & Scalability](#security-and-scalability)  
- [Usage & Examples](#usage-and-examples)  
- [Challenges & Mitigations](#challenges-and-mitigations)  

---


<a id="introduction"></a>
## Introduction
Bitnami's SealedSecrets enables secure secret management in Kubernetes via asymmetric encryption. However, **key rotation** creates operational overhead: older SealedSecrets remain encrypted with deprecated keys unless manually re-encrypted. This proposal automates the re-encryption process by extending the `kubeseal` CLI with a `reencrypt` command.  

### Goals  
- *Zero Manual Effort*: Bulk-reencrypt all SealedSecrets in a cluster or namespace.  
- *Security-First*: Never expose decrypted secrets or private keys.  
- *Scalability*: Handle clusters with thousands of secrets efficiently.  
- *GitOps-Friendly*: Integrate seamlessly with CI/CD pipelines.  

---

<a id="architecture"></a>
## Architecture
The feature extends the `kubeseal` CLI and interacts with Kubernetes APIs.

### High-Level Workflow  
1. *Discover*: List all SealedSecrets in the cluster.  
2. *Decrypt*: Use historical private keys to unlock secrets.  
3. *Re-encrypt*: Seal secrets with the latest public key.  
4. *Update*: Apply modified SealedSecrets back to the cluster.  

---

<a id="component-deep-dive"></a>
## Component Deep Dive

### 1. ClusterScanner  
**Objective**: Fetch all SealedSecret resources.  

**Implementation**:
```diff
// Input: Kubernetes client, namespace (string)
// Output: List of SealedSecret objects ([]v1beta1.SealedSecret)
func ListSealedSecrets(client client.Client, namespace string) ([]v1beta1.SealedSecret, error) {
    opts := []client.ListOption{client.InNamespace(namespace)}
    var sealedSecrets v1beta1.SealedSecretList
    err := client.List(context.Background(), &sealedSecrets, opts...)
    return sealedSecrets.Items, err
}
```
**Reuse**: Leverages the client-go library already used by kubeseal.

---

### 2. KeyManager

**Objective**: Identify decryption keys and fetch the latest public key.

**Implementation**:
```diff
// Input: Controller's key Secret (v1.Secret)
// Output: Map of private keys (map[string]*rsa.PrivateKey), latest public key (*rsa.PublicKey)
func LoadKeys(keySecret *v1.Secret) (map[string]*rsa.PrivateKey, *rsa.PublicKey, error) {
    // Parse private keys from the controller's Secret
    // Sort public keys to identify the latest
    return privateKeys, latestPubKey, nil
}
```
**Reuse**: Adapts key parsing logic from the controller's [main.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/cmd/controller/main.go).

---

### 3. Decryptor

**Objective**: Decrypt a SealedSecret using its original private key.

**Implementation**:
```diff
// Input: SealedSecret (v1beta1.SealedSecret), private key (*rsa.PrivateKey)
// Output: Decrypted Kubernetes Secret (*v1.Secret), error
func DecryptSealedSecret(ss v1beta1.SealedSecret, privateKey *rsa.PrivateKey) (*v1.Secret, error) {
    // Reuse controller's crypto.Decrypt logic
    return decryptedSecret, nil
}
```
**Reuse**: Ports decryption logic from [crypto.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/pkg/crypto/crypto.go).

---

### 4. ReEncryptor

**Objective**: Re-encrypt secrets using the latest public key.

**Implementation**:
```diff
// Input: Secret (v1.Secret), public key (*rsa.PublicKey)
// Output: Re-encrypted SealedSecret (*v1beta1.SealedSecret), error
func ReEncryptSecret(secret v1.Secret, pubKey *rsa.PublicKey) (*v1beta1.SealedSecret, error) {
    // Reuse kubeseal's existing sealing logic
    return sealedSecret, nil
}
```
**Reuse**: Directly calls kubeseal’s Seal function from [client.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/cmd/kubeseal/client.go).

---

### 5. Updater

**Objective**: Apply re-encrypted SealedSecrets to the cluster.

**Implementation**:
```diff
// Input: Kubernetes client, updated SealedSecret (*v1beta1.SealedSecret)
// Output: error
func UpdateSealedSecret(client client.Client, updated *v1beta1.SealedSecret) error {
    return client.Update(context.Background(), updated)
}
```
**Optimization**: Uses server-side apply (`kubectl apply --server-side`) to avoid conflicts.

---

### 6. Orchestrator (`runReencrypt`)

**Workflow**:
```diff
func runReencrypt(namespace string) error {
    // 1. Fetch all SealedSecrets
    secrets, _ := ListSealedSecrets(kubeClient, namespace)

    // 2. Load keys
    privateKeys, latestPubKey, _ := LoadKeys(fetchKeySecret())

    // 3. Process each secret
    for _, ss := range secrets {
        decrypted, _ := DecryptSealedSecret(ss, privateKeys[ss.KeyID])
        reencrypted, _ := ReEncryptSecret(*decrypted, latestPubKey)
        UpdateSealedSecret(kubeClient, reencrypted)
    }
    return nil
}
```

---

<a id="security-and-scalability"></a>
## Security & Scalability

### Security

**RBAC Requirements**:
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
*Key Handling*: Private keys are loaded into memory temporarily and never logged.

### Scalability

- *Concurrency*: Process secrets in parallel with a worker pool (default: 10 goroutines).
- *Batching*: Use pagination (continue tokens) for clusters with 1k+ secrets.
- *Resumability*: Support `--resume-from=<name>` to continue interrupted jobs.

---

<a id="usage-and-examples"></a>
## Usage & Examples

### Prerequisites

- `kubeseal` v0.18.0+ installed
- RBAC permissions granted (see above)

### Command
```bash
kubeseal reencrypt   --all-namespaces   --dry-run   --log-file=report.json
```

### Flags

| Flag          | Purpose                         |
|---------------|----------------------------------|
| `--namespace` | Target a specific namespace     |
| `--dry-run`   | Validate without making changes |
| `--limit=20`  | Control parallel operations     |

### Output
```
2023-10-01 12:00:00 | Re-encrypting 142 SealedSecrets...
2023-10-01 12:00:05 | Success: 140 | Failed: 2
Failed Secrets:
- default/api-token (Error: Invalid private key)
Report saved to report.json.
```

---

<a id="challenges-and-mitigations"></a>
## Challenges & Mitigations

| Challenge              | Mitigation                                                      |
|------------------------|-----------------------------------------------------------------|
| *Private Key Exposure* | Restrict RBAC to read-only for the key Secret.                  |
| *Partial Failures*     | Atomic updates per secret; failed secrets are logged & retried. |
| *Large Clusters*       | Use pagination and limit concurrency to avoid API throttling.   |
| *Legacy Key Formats*   | Skip secrets encrypted with unsupported keys; log warnings.     |

---

## ✅ Conclusion

This implementation provides a secure, automated solution for SealedSecret re-encryption, addressing a critical operational gap in key rotation. By building on proven components and emphasizing scalability, it aligns with modern GitOps workflows while maintaining the security guarantees of sealed-secrets.

**Future Work**:
- Integration with Kubernetes CronJobs for scheduled re-encryption
- Prometheus metrics for monitoring

---

> “Security is a process, not a product.”  
> — *Bruce Schneier*
