+++
author = "Eduard Marbach"
title = "Automate OpenShift Cluster Configuration After Provisioning"
date = "2025-11-11"
description = "Learn how to automate OpenShift cluster configuration using Kustomize and SOPS for secure, repeatable, and version-controlled deployments"
tags = [
    "openshift",
    "kubernetes",
    "kustomize",
    "sops",
    "gitops",
    "infrastructure-as-code",
]
categories = [
    "devops",
    "kubernetes",
]
+++

Provisioning an OpenShift cluster is just the beginning. The real challenge is transforming a freshly deployed cluster with default settings into a production-ready environment that meets your organization's requirements. In this guide, I'll show you how to automate this process using Configuration as Code principles with Kustomize and SOPS.

<!--more-->

## OpenShift Cluster Provisioning Methods

Before diving into automation, let's briefly review the three main ways to provision an OpenShift cluster:

### 1. **Installer-Provisioned Infrastructure (IPI)**
The OpenShift installer fully automates the deployment process, creating both the cluster and the underlying infrastructure (VMs, networks, load balancers, etc.) on supported cloud providers like AWS, Azure, GCP, or on-premises platforms like VMware.

**Pros:** Fastest deployment, fully automated, production-ready architecture  
**Cons:** Less flexibility in infrastructure customization

### 2. **User-Provisioned Infrastructure (UPI)**
You manually provision and configure the underlying infrastructure before running the OpenShift installer. This gives you complete control over network topology, machine sizing, and security configurations.

**Pros:** Maximum flexibility and control, works on any platform  
**Cons:** More complex, requires deep infrastructure knowledge

### 3. **Managed OpenShift Services**
Cloud providers offer managed OpenShift services like Red Hat OpenShift on AWS (ROSA), Azure Red Hat OpenShift (ARO), or OpenShift Dedicated. The provider handles cluster provisioning and management.

**Pros:** Minimal operational overhead, integrated cloud services  
**Cons:** Less control, vendor lock-in, potentially higher costs

## The Post-Deployment Challenge

Regardless of which provisioning method you choose, **every newly deployed OpenShift cluster starts with default configurations**. This means:

- ❌ Default certificates that need replacement
- ❌ No identity providers configured (only kubeadmin exists)
- ❌ Missing custom CAs for corporate proxies
- ❌ No operators installed for storage, backup, virtualization, etc.
- ❌ Default ingress controllers without custom certificates
- ❌ No custom configurations for networking, security, or compliance

**The traditional approach** involves:
1. Logging into the OpenShift Web Console
2. Clicking through multiple UI screens
3. Manually configuring each component
4. Repeating this process for every new cluster
5. Hoping you remembered all the steps correctly

This manual approach is:
- ⏱️ **Time-consuming**: Can take hours or days per cluster
- 🐛 **Error-prone**: Easy to miss steps or make mistakes
- 📝 **Undocumented**: Configuration exists only in someone's head
- 🔄 **Non-repeatable**: Each cluster ends up slightly different
- 🚫 **Not version-controlled**: No audit trail or rollback capability

## The Configuration as Code Solution

**Configuration as Code** treats your cluster configuration the same way you treat application code:

- ✅ **Version controlled** in Git
- ✅ **Reviewed** through pull requests
- ✅ **Tested** before applying to production
- ✅ **Repeatable** across environments
- ✅ **Documented** through the code itself
- ✅ **Auditable** with complete history

By defining your cluster configuration in YAML files and using tools like Kustomize, you can:
1. Apply the same configuration to multiple clusters consistently
2. Track every change with Git history
3. Quickly recover from misconfigurations
4. Share knowledge across teams
5. Automate deployments in CI/CD pipelines

## Tools: Kustomize and SOPS

### Kustomize: Configuration Management

[Kustomize](https://kustomize.io/) is a Kubernetes-native configuration management tool that lets you customize raw YAML files without templates. It's built into `kubectl` and `oc` (OpenShift CLI).

**Key features:**
- **Patches**: Modify existing resources without changing the original files
- **Overlays**: Maintain base configurations with environment-specific overlays
- **Generators**: Create ConfigMaps and Secrets from files or literals
- **Transformers**: Apply common changes (labels, namespaces) across resources

### SOPS: Secret Management

[SOPS (Secrets OPerationS)](https://github.com/mozilla/sops) encrypts YAML, JSON, and other file formats while keeping the structure readable. It integrates with key management services like AWS KMS, Azure Key Vault, or GCP KMS.

**Key features:**
- **Selective encryption**: Only encrypt sensitive fields, keep structure visible
- **Git-friendly**: Encrypted files can be safely committed to version control
- **Multiple key backends**: AWS KMS, Azure Key Vault, GCP KMS, PGP, age
- **Audit trail**: Track who encrypted/decrypted what and when

Together with **ksops** (a Kustomize plugin for SOPS), you can manage secrets alongside regular configurations.

## Project Structure

Here's the structure for our OpenShift configuration repository:

```
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── proxy-cluster.yaml
│   └── ingresscontroller-default.yaml
└── overlays/
    └── my_site/
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── apiserver-cluster.yaml
        ├── oauth.yaml
        ├── custom-ca-configmap.enc.yaml
        ├── cert-api-secret.enc.yaml
        ├── cert-wildcard-secret.enc.yaml
        ├── oauth-secret.enc.yaml
        ├── secret-generator.yaml
        └── subscriptions/
            ├── sub-oadp.yaml
            ├── sub-lvms.yaml
            ├── sub-nmstate.yaml
            └── sub-kubevirt-hyperconverged.yaml
```

**Base layer**: Contains common configurations shared across all clusters  
**Overlay layer**: Contains site-specific configurations and secrets

## Configuration Examples

### Base Configuration

The base layer defines resources that apply to all clusters:

#### proxy-cluster.yaml
```yaml
apiVersion: config.openshift.io/v1
kind: Proxy
metadata:
  name: cluster
spec:
  trustedCA:
    name: custom-ca
```

This configures OpenShift to trust your corporate CA certificates.

#### ingresscontroller-default.yaml
```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  defaultCertificate:
    name: cert-wildcard
```

This replaces the default self-signed certificate with your custom wildcard certificate.

#### base/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: openshift-certificates-base

resources:
  - proxy-cluster.yaml
  - ingresscontroller-default.yaml

labels:
  - includeSelectors: true
    pairs:
      de.db/team: responsible
      managed-by: kustomize
```

### Overlay Configuration

The overlay adds site-specific configurations:

#### oauth.yaml
```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - mappingMethod: add
      name: Azure
      openID:
        claims:
          email:
            - email
          name:
            - name
          preferredUsername:
            - customfieldifavailable
            - preferred_username
            - email
        clientID: 123-123-123
        clientSecret:
          name: openid-client-secret
        extraScopes: []
        issuer: https://login.microsoftonline.com/123123123
      type: OpenID
```

This configures Azure AD as an identity provider for OpenShift authentication.

#### Operator Subscriptions

Install and configure operators like OADP (backup), LVMS (storage), NMState (networking), and KubeVirt (virtualization):

```yaml
# subscriptions/sub-oadp.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/redhat-oadp-operator.openshift-adp: ""
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable-1.4
  installPlanApproval: Manual
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: oadp-operator.v1.4.6
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-adp-7nbcd
  namespace: openshift-adp
spec:
  targetNamespaces:
    - openshift-adp
```

#### overlays/my_site/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: test-openshift

resources:
  - ../../base
  - apiserver-cluster.yaml
  - oauth.yaml
  - ./subscriptions/sub-oadp.yaml
  - ./subscriptions/sub-lvms.yaml
  - ./subscriptions/sub-nmstate.yaml
  - ./subscriptions/sub-kubevirt-hyperconverged.yaml
  - namespace.yaml

labels:
  - includeSelectors: true
    pairs:
      de.db/site: mysite
      de.db/team: responsible

generators:
  - ./secret-generator.yaml
```

## Managing Secrets with SOPS

### SOPS Configuration

Create a `.sops.yaml` file to configure encryption rules:

```yaml
creation_rules:
  - unencrypted_regex: "^(apiVersion|metadata|kind|type)$"
    path_regex: (.enc.yaml|.enc.yml)
    key_groups:
      - kms:
          - arn: arn:aws:kms:eu-central-1:123456789:key/abc123def456
```

**Key points:**
- `unencrypted_regex`: Keep these fields unencrypted for Kubernetes to read the resource type
- `path_regex`: Only encrypt files ending in `.enc.yaml` or `.enc.yml`
- `kms`: Use AWS KMS for encryption (you can also use Azure Key Vault, GCP KMS, or age)

### Encrypting Secrets

Create your secret file with sensitive data:

```yaml
# cert-api-secret.enc.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cert-api
  namespace: openshift-config
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAKZ...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0BA...
    -----END PRIVATE KEY-----
```

Encrypt the file with SOPS:

```bash
sops -e -i cert-api-secret.enc.yaml
```

After encryption, the file looks like:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cert-api
  namespace: openshift-config
type: kubernetes.io/tls
stringData:
  tls.crt: ENC[AES256_GCM,data:8h5jW...,tag:abc123==]
  tls.key: ENC[AES256_GCM,data:Kj9mP...,tag:def456==]
sops:
  kms:
    - arn: arn:aws:kms:eu-central-1:123456789:key/abc123def456
      created_at: "2025-11-11T10:30:00Z"
      enc: AQICAHh...
  # ... more sops metadata
```

**Notice:** The structure remains readable (great for Git diffs!), but sensitive values are encrypted.

### Secret Generator Configuration

Configure ksops to decrypt secrets during Kustomize build:

```yaml
# secret-generator.yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: example-secret-generator
  annotations:
    config.kubernetes.io/function: |
      exec:
        path: ksops
files:
  - ./cert-api-secret.enc.yaml
  - ./cert-wildcard-secret.enc.yaml
  - ./custom-ca-configmap.enc.yaml
  - ./oauth-secret.enc.yaml
```

## Deployment Process

### Prerequisites

1. **Install required tools:**
   ```bash
   # Install kustomize
   brew install kustomize  # macOS
   # or download from https://kubectl.docs.kubernetes.io/installation/kustomize/
   
   # Install SOPS
   brew install sops  # macOS
   # or download from https://github.com/mozilla/sops/releases
   
   # Install ksops
   # Follow instructions at https://github.com/viaduct-ai/kustomize-sops
   ```

2. **Configure AWS credentials** (if using AWS KMS):
   ```bash
   export AWS_PROFILE=your-profile
   # or
   export AWS_ACCESS_KEY_ID=...
   export AWS_SECRET_ACCESS_KEY=...
   ```

3. **Login to OpenShift:**
   ```bash
   oc login https://api.your-cluster.com:6443
   ```

### Step 1: Initial Deployment

Apply the configuration to your cluster:

```bash
# Build and preview the resources
kustomize build --enable-alpha-plugins kustomize/overlays/my_site | less

# Apply the configuration
kustomize build --enable-alpha-plugins kustomize/overlays/my_site | oc apply -f -
```

### Step 2: Handle CRD Installation Failures

**Important:** Some resources will fail initially because the Custom Resource Definitions (CRDs) don't exist yet. This is expected behavior when installing operators.

**What happens:**
1. Operator Subscriptions create InstallPlans
2. InstallPlans install CRDs and operators
3. Custom resources (like `LVMCluster`) fail because CRDs aren't ready yet

**Example error:**
```
error: unable to recognize "STDIN": no matches for kind "LVMCluster" 
in version "lvm.topolvm.io/v1alpha1"
```

### Step 3: Approve InstallPlans

For security, operator installations use `installPlanApproval: Manual`, requiring manual approval:

1. **Open the OpenShift Web Console**
2. **Navigate to:** Operators → Installed Operators
3. **For each operator** (OADP, LVMS, NMState, KubeVirt):
   - Click on the operator namespace
   - Find the InstallPlan in "Pending" state
   - Click "Approve"
4. **Wait** for operators to finish installing (watch the status change to "Succeeded")

Alternatively, approve via CLI:

```bash
# List pending install plans
oc get installplan -A | grep -v "true"

# Approve a specific install plan
oc patch installplan <install-plan-name> -n <namespace> \
  --type merge --patch '{"spec":{"approved":true}}'
```

### Step 4: Re-apply Configuration

After operators are installed and CRDs are available:

```bash
# Apply again to create custom resources
kustomize build --enable-alpha-plugins kustomize/overlays/my_site | oc apply -f -
```

This time, resources like `LVMCluster`, `DataProtectionApplication`, etc., will be created successfully.

### Step 5: Verify Deployment

Check that everything is running:

```bash
# Check operator installations
oc get csv -A

# Check custom resources
oc get lvmcluster -n openshift-storage
oc get dataprotectionapplication -n openshift-adp

# Verify OAuth configuration
oc get oauth cluster -o yaml

# Check ingress controller
oc get ingresscontroller default -n openshift-ingress-operator -o yaml

# Verify certificates
oc get secret -n openshift-config | grep cert
```

## Best Practices

### 1. **Use Git for Version Control**
```bash
git add kustomize/
git commit -m "Add OpenShift cluster configuration"
git push
```

### 2. **Separate Secrets from Configuration**
- Keep all `.enc.yaml` files encrypted
- Never commit unencrypted secrets to Git
- Use different KMS keys for different environments

### 3. **Use Overlays for Environments**
```
overlays/
├── dev/
├── staging/
└── production/
```

### 4. **Document Your Configurations**
- Add comments explaining non-obvious configurations
- Maintain a README with deployment instructions
- Document any manual steps required

### 5. **Test in Lower Environments First**
- Deploy to dev/test clusters first
- Validate configurations before promoting to production
- Use GitOps tools like ArgoCD for automated deployments

### 6. **Regular Backups**
```bash
# Backup current cluster configuration
oc get all,secrets,configmaps -A -o yaml > cluster-backup-$(date +%Y%m%d).yaml
```

### 7. **Use Labels Consistently**
Apply meaningful labels for tracking and organization:
```yaml
labels:
  - includeSelectors: true
    pairs:
      app: myapp
      environment: production
      team: platform
      managed-by: kustomize
```

## Troubleshooting

### SOPS Decryption Fails
```bash
# Error: Failed to get the data key required to decrypt the SOPS file

# Solution: Check AWS credentials and KMS key permissions
aws sts get-caller-identity
aws kms describe-key --key-id <your-kms-key-id>
```

### Kustomize Build Fails
```bash
# Error: accumulating resources: accumulation err='accumulating resources...'

# Solution: Validate YAML syntax
yamllint kustomize/overlays/my_site/*.yaml

# Check kustomization.yaml references
kustomize build --enable-alpha-plugins kustomize/overlays/my_site
```

### Operator Installation Hangs
```bash
# Check operator pod logs
oc logs -n <operator-namespace> -l name=<operator-name>

# Check InstallPlan status
oc get installplan -n <namespace> -o yaml

# Check for resource conflicts
oc get events -n <namespace> --sort-by='.lastTimestamp'
```

## Advanced: GitOps Integration

For production environments, integrate with GitOps tools like ArgoCD:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: openshift-config
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/openshift-config
    targetRevision: main
    path: kustomize/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

This enables:
- **Automatic sync** when configurations change in Git
- **Drift detection** when manual changes are made
- **Rollback capability** to previous versions
- **Visual dashboard** for configuration status

## Conclusion

Automating OpenShift cluster configuration with Kustomize and SOPS transforms cluster management from a manual, error-prone process into a repeatable, version-controlled workflow. By embracing Configuration as Code principles, you gain:

- ✅ **Consistency** across multiple clusters
- ✅ **Repeatability** for disaster recovery and new deployments
- ✅ **Security** through encrypted secrets management
- ✅ **Auditability** with complete Git history
- ✅ **Collaboration** through code reviews and pull requests
- ✅ **Speed** in deploying and updating configurations

The initial investment in setting up this automation pays dividends every time you need to deploy a new cluster, update configurations, or recover from issues.

## Complete Example Repository

All the examples and configurations shown in this post are available in my GitHub repository:

**[blog-20251111-openshift-config](https://github.com/BlackDark/blog-20251111-openshift-config)**

The repository includes:
- Complete Kustomize base and overlay configurations
- SOPS encrypted secrets examples
- Operator subscription definitions
- Ready-to-use `.sops.yaml` configuration

Feel free to use it as a template for your own OpenShift automation!

## Resources

- [OpenShift Documentation](https://docs.openshift.com/)
- [Kustomize Documentation](https://kustomize.io/)
- [SOPS GitHub Repository](https://github.com/mozilla/sops)
- [ksops - Kustomize SOPS Plugin](https://github.com/viaduct-ai/kustomize-sops)
- [OpenShift Operators Hub](https://operatorhub.io/)

---

Have you automated your OpenShift cluster configurations? What challenges did you face? Share your experiences in the comments!
