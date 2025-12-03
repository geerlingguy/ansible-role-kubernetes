# Ansible Role: Kubernetes

[![CI](https://github.com/geerlingguy/ansible-role-kubernetes/actions/workflows/ci.yml/badge.svg)](https://github.com/geerlingguy/ansible-role-kubernetes/actions/workflows/ci.yml)

An Ansible Role that installs [Kubernetes](https://kubernetes.io) on Linux.

## Requirements

Requires a compatible [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes); recommended role for CRI installation: `geerlingguy.containerd`.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
kubernetes_packages:
  - name: kubelet
    state: present
  - name: kubectl
    state: present
  - name: kubeadm
    state: present
  - name: kubernetes-cni
    state: present
```

Kubernetes packages to be installed on the server. You can either provide a list of package names, or set `name` and `state` to have more control over whether the package is `present`, `absent`, `latest`, etc.

```yaml
kubernetes_version: '1.33'
kubernetes_version_rhel_package: '1.33'
```

The minor version of Kubernetes to install. The plain `kubernetes_version` is used to pin an apt package version on Debian, and as the Kubernetes version passed into the `kubeadm init` command (see `kubernetes_version_kubeadm`). The `kubernetes_version_rhel_package` variable must be a specific Kubernetes release, and is used to pin the version on Red Hat / CentOS servers.

```yaml
kubernetes_role: control_plane
```

Whether the particular server will serve as a Kubernetes `control_plane` (default) or `node`. The control plane will have `kubeadm init` run on it to intialize the entire K8s control plane, while `node`s will have `kubeadm join` run on them to join them to the `control_plane`.

### Variables to configure kubeadm and kubelet with `kubeadm init` through a config file (recommended)

With this role, `kubeadm init` will be run with `--config <FILE>`.

```yaml
kubernetes_kubeadm_kubelet_config_file_path: '/etc/kubernetes/kubeadm-kubelet-config.yaml'
```

Path for `<FILE>`. If the directory does not exist, this role will create it.

The following variables are parsed as options to <FILE>. To understand its syntax, see [kubelet-integration](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration) and [kubeadm-config-file](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file) . The skeleton (`apiVersion`, `kind`) of the config file will be created by this role, so do not define them within the variables. (See `templates/kubeadm-kubelet-config.j2`).

```yaml
kubernetes_config_init_configuration:
  localAPIEndpoint:
    advertiseAddress: "{{ kubernetes_apiserver_advertise_address | default(ansible_default_ipv4.address, true) }}"
```

Defines the options under `kind: InitConfiguration`. Including `kubernetes_apiserver_advertise_address` here is for backward-compatibilty to older versions of this role, where `kubernetes_apiserver_advertise_address` was used with a command-line-option.

```yaml
kubernetes_config_cluster_configuration:
  networking:
    podSubnet: "{{ kubernetes_pod_network.cidr }}"
  kubernetesVersion: "{{ kubernetes_version_kubeadm }}"
```

Options under `kind: ClusterConfiguration`. Including `kubernetes_pod_network.cidr` and `kubernetes_version_kubeadm` here are for backward-compatibilty to older versions of this role, where they were used with command-line-options.

```yaml
kubernetes_config_kubelet_configuration:
  cgroupDriver: systemd
```

Options to configure kubelet on any nodes in your cluster through the `kubeadm init` process. For syntax options read the [kubelet config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file) and [kubelet integration](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration) documentation.

NOTE: This is the recommended way to do the kubelet-configuration. Most command-line-options are deprecated.

NOTE: The recommended cgroupDriver depends on your [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes). When using this role with Docker instead of containerd, this value should be changed to `cgroupfs`.

```yaml
kubernetes_config_kube_proxy_configuration: {}
```

Options to configure kubelet's proxy configuration in the `KubeProxyConfiguration` section of the kubelet configuration.

### Variables to configure kubeadm and kubelet through command-line-options

```yaml
kubernetes_kubelet_extra_args: ""
kubernetes_kubelet_extra_args_config_file: /etc/default/kubelet
```

Extra args to pass to `kubelet` during startup. E.g. to allow `kubelet` to start up even if there is swap is enabled on your server, set this to: `"--fail-swap-on=false"`. Or to specify the node-ip advertised by `kubelet`, set this to `"--node-ip={{ ansible_host }}"`. **This option is deprecated. Please use `kubernetes_config_kubelet_configuration` instead.**

```yaml
kubernetes_kubeadm_init_extra_opts: ""
```

Extra args to pass to `kubeadm init` during K8s control plane initialization. E.g. to specify extra Subject Alternative Names for API server certificate, set this to: `"--apiserver-cert-extra-sans my-custom.host"`

```yaml
kubernetes_join_command_extra_opts: ""
```

Extra args to pass to the generated `kubeadm join` command during K8s node initialization. E.g. to ignore certain preflight errors like swap being enabled, set this to: `--ignore-preflight-errors=Swap`

### Additional variables

```yaml
kubernetes_allow_pods_on_control_plane: true
```

Whether to remove the taint that denies pods from being deployed to the Kubernetes control plane. If you have a single-node cluster, this should definitely be `True`. Otherwise, set to `False` if you want a dedicated Kubernetes control plane which doesn't run any other pods.

```yaml
kubernetes_pod_network:
  # Flannel CNI.
  cni: 'flannel'
  cidr: '10.244.0.0/16'
  #
  # Calico CNI.
  # cni: 'calico'
  # cidr: '192.168.0.0/16'
  #
  # Weave CNI.
  # cni: 'weave'
  # cidr: '192.168.0.0/16'
```

This role currently supports `flannel` (default), `calico` or `weave` for cluster pod networking. Choose only one for your cluster; converting between them is not done automatically and could result in broken networking; if you need to switch from one to another, it should be done outside of this role.

```yaml
kubernetes_apiserver_advertise_address: ''`
kubernetes_version_kubeadm: 'stable-{{ kubernetes_version }}'`
kubernetes_ignore_preflight_errors: 'all'
```

Options passed to `kubeadm init` when initializing the Kubernetes control plane. The `kubernetes_apiserver_advertise_address` defaults to `ansible_default_ipv4.address` if it's left empty.

```yaml
kubernetes_apt_release_channel: "stable"
kubernetes_apt_repository: "https://pkgs.k8s.io/core:/{{ kubernetes_apt_release_channel }}:/v{{ kubernetes_version }}/deb/"
```

Apt repository options for Kubernetes installation.

```yaml
kubernetes_yum_base_url: "https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_version }}/rpm/"
kubernetes_yum_gpg_key: "https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_version }}/rpm/repodata/repomd.xml.key"
kubernetes_yum_gpg_check: true
kubernetes_yum_repo_gpg_check: true
```

Yum repository options for Kubernetes installation. You can change `kubernete_yum_gpg_key` to a different url if you are behind a firewall or provide a trustworthy mirror. Usually in combination with changing `kubernetes_yum_base_url` as well.

```yaml
kubernetes_flannel_manifest_file: https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Flannel manifest file to apply to the Kubernetes cluster to enable networking. You can copy your own files to your server and apply them instead, if you need to customize the Flannel networking configuration.

```yaml
kubernetes_calico_manifest_file: https://projectcalico.docs.tigera.io/manifests/calico.yaml
```

Calico manifest file to apply to the Kubernetes cluster (if using Calico instead of Flannel).

## Dependencies

None.

## Example Playbooks

### Single node (control-plane-only) cluster

```yaml
- hosts: all

  vars:
    kubernetes_allow_pods_on_control_plane: true

  roles:
    - geerlingguy.docker
    - geerlingguy.kubernetes
```

### Two or more nodes (single control-plane) cluster

Control plane inventory vars:

```yaml
kubernetes_role: "control_plane"
```

Node(s) inventory vars:

```yaml
kubernetes_role: "node"
```

Playbook:

```yaml
- hosts: all

  vars:
    kubernetes_allow_pods_on_control_plane: true

  roles:
    - geerlingguy.docker
    - geerlingguy.kubernetes
```

### IPv6 only two or more nodes (single control-plane) cluster using containerd

Download default config file for flannel from https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml and store it in files/kube-flannel-config.yaml next to ansible scripts.

Update net-conf.json in property:

```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    k8s-app: flannel
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: flannel
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: flannel
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: flannel
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    k8s-app: flannel
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "EnableIPv4": false,
      "EnableIPv6": true,
      "IPv6Network": "fd76:cac2:6150::/56",
      "IPv6Subnet": "fd76:cac2:6150::1/64",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
    k8s-app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: ghcr.io/flannel-io/flannel:v0.27.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: ghcr.io/flannel-io/flannel:v0.27.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        - name: CONT_WHEN_CACHE_NOT_READY
          value: "false"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

Control plane inventory vars:

```yaml
kubernetes_role: "control_plane"
```

Node(s) inventory vars:

```yaml
kubernetes_role: "node"
```

Playbook:

```yaml

- hosts: all

  vars:
    kubernetes_flannel_manifest_file: '/etc/kubernetes/kube-flannel-config.yaml'

  tasks:
    - name: Create the directory for the kubernetes_config_file
      ansible.builtin.file:
        path: "{{ kubernetes_flannel_manifest_file | dirname }}"
        state: directory

    - name: Copy 'kube-flannel-config.yaml'
      ansible.builtin.copy:
        src: files/kube-flannel-config.yaml
        dest: "{{ kubernetes_flannel_manifest_file }}"
        owner: root
        group: root
        mode: u+rw

- hosts: all

  vars:
    containerd_config_cgroup_driver_systemd: true
    kubernetes_allow_pods_on_control_plane: true
    kubernetes_flannel_manifest_file: '/etc/kubernetes/kube-flannel-config.yaml'
    kubernetes_pod_network:
      cni: 'flannel'
      cidr: 'fd76:cac2:6150::/56'
      serviceCidr: 'fd76:cac2:6150:1::/112'

  roles:
    - geerlingguy.containerd
    - geerlingguy.kubernetes
```

Then, log into the Kubernetes control plane, and run `kubectl get nodes` as root, and you should see a list of all the servers.

## License

MIT / BSD

## Author Information

This role was created in 2018 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
