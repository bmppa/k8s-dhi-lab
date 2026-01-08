# 🧪 Enforcing Docker Hardened Images with Validating Admission Policies

## 🎯 Lab Goal

Demonstrate how to **enforce the use of Docker Hardened Images (DHI)** in Kubernetes using **Validating Admission Policies (VAP)**. The lab will:

* Allow only approved Docker Hardened Images
* Block non-hardened or unknown images at admission time
* Provide clear denial messages for developers

This lab is **vendor-neutral at the Kubernetes layer** and aligns with **supply chain security best practices**.

---

## Learning Outcomes

By completing this lab, you will:

* Understand Kubernetes Validating Admission Policies
* Enforce hardened images *before runtime*
* Demonstrate supply chain controls to security teams

---

## Prerequisites

* Kubernetes **v1.26+** (VAP is GA)
* `kubectl` access with cluster-admin
* A cluster with admission policies enabled (default in recent distros)
* Familiarity with Docker Hardened Images naming

---

## Threat Model

**Risk:** Developers deploy images that are:

* Unverified
* Built from insecure base images
* Missing hardening controls

**Control:** Enforce image provenance and trust *before* workloads run.

---

## Assumption: Docker Hardened Image Pattern

For this lab, we assume approved images follow one of these patterns:

* `dhi.io/`

> You can later replace this with **digest pinning**, **Cosign verification**, or **SBOM checks**.

---

## Step 1 – Create a Test Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab
```

---

## Step 2 – Validating Admission Policy

This policy **denies pods** that do not use approved Docker Hardened Images.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-image-registry
spec:
  failurePolicy: Fail
  matchConstraints:    
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: "object.spec.containers.all(c, c.image.startsWith('dhi.io/'))"
      message: "Only Docker Hardened Images are allowed in this cluster"
```

---

## Step 3 – Bind the Policy

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-image-registry-binding
spec:
  policyName: require-image-registry
  validationActions: ["Deny"]
```

---

## Step 4 – Deploy a Non-Compliant Pod (Expected: ❌ Denied)

```
kubectl run pod --image nginx
```

### Expected Result

```text
The pods "pod" is invalid: : ValidatingAdmissionPolicy 'require-image-registry' with binding 'require-image-registry-binding' denied request: Only Docker Hardened Images are allowed in this cluster
```

---

## Step 5 – Deploy a Hardened Image (Expected: ✅ Allowed)

Update the flag `--from-file` with the correct path.

```
kubectl create secret docker-registry my-secret --from-file=/path/to/.docker/config.json -o yaml > my-secret.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: lab
spec:
  containers:
  - name: my-dhi-container
    image: dhi.io/nginx:1
  imagePullSecrets:
  - name: my-secret
```

---

## Step 6 – Test Update Enforcement

Try updating an existing pod to a non-hardened image:

```bash
kubectl set image pod/my-pod my-dhi-container=nginx:latest -n lab
```

### Expected Result

* Update is **denied**
* Policy enforces drift prevention

```text
error: failed to patch image update to pod template: pods "my-pod" is forbidden: ValidatingAdmissionPolicy 'require-image-registry' with binding 'require-image-registry-binding' denied request: Only Docker Hardened Images are allowed in this cluster
```

---

## Step 7 – Observability & Audit

Admission decisions appear in:

* Kubernetes API server audit logs

**Key signal:** `admission.k8s.io/validation_failure`

---

## Comparison to Other Controls

| Control        | Strength                  | Limitation               |
| -------------- | ------------------------- | ------------------------ |
| VAP            | Native, fast, declarative | No deep image inspection |
| OPA/Gatekeeper | Rich logic                | Extra control plane      |
| Runtime tools  | Detects abuse             | Too late for prevention  |
