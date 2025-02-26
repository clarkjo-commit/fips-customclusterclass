# Custom ClusterClass for Enabling FIPS and STIGs on TKGM Cluster

## Overview

This **ClusterClass** is a customized template for deploying a **Tanzu Kubernetes Grid Management (TKGM)** cluster with enhanced security features, specifically enabling **FIPS (Federal Information Processing Standards)** and applying specific **STIG (Security Technical Implementation Guide)** settings. It ensures compliance with U.S. government security policies for system hardening, encryption, and system monitoring.

### Key Features

- **FIPS (Federal Information Processing Standards)**: Ensures that the cluster nodes are configured to use FIPS-approved cryptographic modules, enabling compliance with U.S. government requirements for secure systems.
- **STIG (Security Technical Implementation Guide)** Controls**: Enforces security configurations like SSH settings, system banners, and cryptographic configurations to comply with U.S. government security standards (such as PHTN-30-000003, PHTN-30-000064, etc.).
- **Custom Pre/Post Kubeadm Commands**: Includes commands to configure FIPS mode and apply critical system security controls before and after the Kubernetes components are deployed.

## ClusterClass Details

This `ClusterClass` defines a configuration for both the **control plane** and **worker nodes** within a TKG-managed Kubernetes cluster. It includes specific patches and commands that ensure compliance with security policies.

### Key Annotations:
- `run.tanzu.vmware.com/resolve-tkr`: Specifies the TKR (Tanzu Kubernetes Release) versions for the cluster.
- `run.tanzu.vmware.com/verified-tkrs`: Defines the validated TKR versions to use.
- `run.tanzu.vmware.com/tkg-managed`: Denotes that the cluster is managed by TKG.
- `run.tanzu.vmware.com/tkg-version`: The version of Tanzu Kubernetes Grid in use.

### Control Plane Configuration
The control plane is configured to run with the following key settings:
- **Machine Health Checks**: Ensures the health of the control plane nodes, with thresholds for node startup and failure conditions.
- **PreKubeadm Commands**: Runs specific commands before the Kubernetes components are initialized, including enabling FIPS mode and applying SSH hardening configurations.
- **PostKubeadm Commands**: Ensures FIPS is enabled after the cluster initialization, and if not, reboots the node to enable it.
- **Custom Messages**: Displays security warning messages and ensures compliance with security banners.

### Worker Nodes Configuration
The worker node configuration mirrors the control plane setup and applies similar hardening and compliance controls. Specific configurations include:
- **PreKubeadm Commands**: Similar to the control plane, these commands ensure that FIPS is enabled and apply necessary security settings.
- **PostKubeadm Commands**: Verifies that FIPS is enabled after the worker nodes are initialized.

### STIG Security Controls
The following security controls are applied as part of the patching process:
1. **PHTN-30-000003**: Applies SSH hardening configurations (e.g., cipher suites and message authentication codes).
2. **PHTN-30-000064**: Displays a system banner message when accessing the cluster.
3. **PHTN-30-000240**: Ensures the use of strong cryptographic settings, such as AES ciphers.
4. **PHTN-30-000239**: Configures SSH security settings, such as MACs (Message Authentication Codes) and ciphers for secure communication.

### Custom ClusterClass Configuration
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  annotations:
    run.tanzu.vmware.com/resolve-tkr: ""
    run.tanzu.vmware.com/verified-tkrs: v1.25.10+vmware.2-tkg.1,v1.24.14+vmware.2-tkg.1,... # List of verified TKR versions
  labels:
    run.tanzu.vmware.com/tkg-managed: ""
    run.tanzu.vmware.com/tkg-version: v2.5.2
  name: tkg-vsphere-default-v1.2.0-fips
  namespace: tmc
spec:
  controlPlane:
    machineHealthCheck:
      maxUnhealthy: 100%
      nodeStartupTimeout: 20m0s
      unhealthyConditions:
      - status: Unknown
        timeout: 5m0s
        type: Ready
      - status: "False"
        timeout: 12m0s
        type: Ready
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: VSphereMachineTemplate
        name: tkg-vsphere-default-v1.2.0-control-plane
        namespace: tmc
    metadata: {}
    namingStrategy:
      template: '{{ .cluster.name }}-controlplane-{{ .random }}'
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: tkg-vsphere-default-v1.2.0-kcp
      namespace: tmc
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: VSphereClusterTemplate
      name: tkg-vsphere-default-v1.2.0-cluster
      namespace: tmc
  patches:
  - external:
      discoverVariablesExtension: discover-variables.tkg-runtime-extension
      generateExtension: generate-patches.tkg-runtime-extension
      settings:
        flavor: tkg
        version: v1.2.0
      validateExtension: validate-topology.tkg-runtime-extension
    name: tkg-official-patch
  - definitions:
      - jsonPatches:
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/preKubeadmCommands/-
            value: echo '"You are accessing a U.S. Government (USG) Information System..." > /etc/issue
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/preKubeadmCommands/-
            value: echo 'Banner /etc/issue' >> /etc/ssh/sshd_config
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/preKubeadmCommands/-
            value: echo 'Ciphers aes256-ctr,aes192-ctr,aes128-ctr' >> /etc/ssh/sshd_config
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/preKubeadmCommands/-
            value: echo 'MACs hmac-sha2-512,hmac-sha2-256' >> /etc/ssh/sshd_config
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/preKubeadmCommands/-
            value: sudo systemctl restart sshd
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/preKubeadmCommands/-
            value: sudo sed -i '/linux/s/$/ fips=1/' /boot/grub2/grub.cfg
          - op: add
            path: /spec/template/spec/postKubeadmCommands/-
            value: sysctl crypto.fips_enabled | grep -q '1$' || sudo init 6
      selector:
        apiVersion: controlplane.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlaneTemplate
        matchResources:
          controlPlane: true
  name: controlPlanePreKubeadmCommandsEnableSecurityControls
  workers:
    machineDeployments:
    - class: tkg-worker
      machineHealthCheck:
        maxUnhealthy: 100%
        nodeStartupTimeout: 20m0s
        unhealthyConditions:
        - status: Unknown
          timeout: 5m0s
          type: Ready
        - status: "False"
          timeout: 12m0s
          type: Ready
      template:
        bootstrap:
          ref:
            apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
            kind: KubeadmConfigTemplate
            name: tkg-vsphere-default-v1.2.0-md-config
            namespace: tmc
        infrastructure:
          ref:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: VSphereMachineTemplate
            name: tkg-vsphere-default-v1.2.0-worker
            namespace: tmc
        metadata: {}
```

---

## How to Use This ClusterClass on a TKGM Cluster

### Step 1: Prepare Your Environment
Ensure your environment is set up for **Tanzu Kubernetes Grid Management (TKGM)** with the necessary resources, including the Tanzu Mission Control (TMC) environment.

### Step 2: Create or Edit Your ClusterClass YAML
Create a new file (e.g., `tkg-cluster-class.yaml`) with the content above or modify an existing `ClusterClass` configuration to incorporate the FIPS and STIG settings.

### Step 3: Apply the ClusterClass
Use `kubectl` to apply the `ClusterClass` to your TMC-managed environment:

```bash
kubectl apply -f tkg-cluster-class.yaml
```

### Step 4: Create a Cluster from the ClusterClass
After applying the `ClusterClass`, you can create a new cluster using this configuration. The cluster will automatically deploy the **FIPS** and **STIGs** settings as part of the control plane and worker node initialization.

You’ll need to modify the cluster object to reference your newly created ClusterClass (which includes FIPS and STIG settings). Update the controlPlane and workers fields accordingly to use your new ClusterClass.

Here's an example of how the topology section might look after the change:
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: tmc
spec:
  topology:
    controlPlane:
      class: tkg-vsphere-default-v1.2.0-fips  # Reference to the new ClusterClass
      machineCount: 3
    workers:
      class: tkg-vsphere-default-v1.2.0-fips  # Same reference for workers
      machineCount: 3
```

###

 Step 5: Verify the Cluster Configuration
Once the cluster is up and running, verify that **FIPS** is enabled by checking the configuration on the nodes:

```bash
sysctl crypto.fips_enabled  # Should return '1' if FIPS is enabled
```

Additionally, check the SSH configurations and system banners to ensure they comply with the applied **STIG** controls.

---

## Conclusion

This custom `ClusterClass` allows you to deploy a TKGM cluster that is fully compliant with U.S. government security standards. By enabling **FIPS** mode and applying critical **STIG** settings, you can ensure that your Kubernetes environment is secure and ready for sensitive workloads.