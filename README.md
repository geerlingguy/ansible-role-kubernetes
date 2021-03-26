# Ansible Role: Kubernetes

[![CI](https://github.com/geerlingguy/ansible-role-kubernetes/workflows/CI/badge.svg?event=push)](https://github.com/geerlingguy/ansible-role-kubernetes/actions?query=workflow%3ACI)

An Ansible Role that installs [Kubernetes](https://kubernetes.io) on Linux.

## Requirements

Requires Docker or another [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes) ; recommended role for Docker installation: `geerlingguy.docker`.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    kubernetes_packages:
      - name: kubelet
        state: present
      - name: kubectl
        state: present
      - name: kubeadm
        state: present
      - name: kubernetes-cni
        state: present

Kubernetes packages to be installed on the server. You can either provide a list of package names, or set `name` and `state` to have more control over whether the package is `present`, `absent`, `latest`, etc.

    kubernetes_version: '1.20'
    kubernetes_version_rhel_package: '1.20.4'

The minor version of Kubernetes to install. The plain `kubernetes_version` is used to pin an apt package version on Debian, and as the Kubernetes version passed into the `kubeadm init` command (see `kubernetes_version_kubeadm`). The `kubernetes_version_rhel_package` variable must be a specific Kubernetes release, and is used to pin the version on Red Hat / CentOS servers.

    kubernetes_role: master

Whether the particular server will serve as a Kubernetes `master` (default) or `node`. The master will have `kubeadm init` run on it to intialize the entire K8s control plane, while `node`s will have `kubeadm join` run on them to join them to the `master`.

### Variables to configure kubeadm and kubelet with `kubeadm init` through a config file (recommended)

With this role, `kubeadm init` will be run with `--config <FILE>`.

    kubernetes_kubeadm_kubelet_config_file_path: '/etc/kubernetes/kubeadm-kubelet-config.yaml'

Path for `<FILE>`. If the directory does not exist, this role will create it.

The following variables are parsed as options to <FILE>. To understand its syntax, see https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration and https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file . The skeleton (`apiVersion`, `kind`) of the config file will be created by this role, so do not define them within the variables. (See `templates/kubeadm-kubelet-config.yaml`).

    kubernetes_config_init_configuration:
      localAPIEndpoint:
        advertiseAddress: "{{ kubernetes_apiserver_advertise_address | default(ansible_default_ipv4.address, true) }}"

Defines the options under `kind: InitConfiguration`. Including `kubernetes_apiserver_advertise_address` here is for backward-compatibilty to older versions of this role, where `kubernetes_apiserver_advertise_address` was used with a command-line-option.

    kubernetes_config_cluster_configuration:
      networking:
        podSubnet: "{{ kubernetes_pod_network.cidr }}"
      kubernetesVersion: "{{ kubernetes_version_kubeadm }}"

Options under `kind: ClusterConfiguration`. Including `kubernetes_pod_network.cidr` and `kubernetes_version_kubeadm` here are for backward-compatibilty to older versions of this role, where they were used with command-line-options.

    kubernetes_config_kubelet_configuration:
      cgroupDriver: cgroupfs

Options to configure kubelet on any nodes in your cluster through the `kubeadm init` process. To get the syntax of this options see https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file and https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration.

NOTE: This is the recommended way to do the kubelet-configuration. Most command-line-options are deprecated.

NOTE: The recommended cgroupDriver depends on your [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes). When using this role with containerd instead of docker, this value should be changed to `systemd`.

### Variables to configure kubeadm and kubelet through command-line-options

    kubernetes_kubelet_extra_args: ""
    kubernetes_kubelet_extra_args_config_file: /etc/default/kubelet

Extra args to pass to `kubelet` during startup. E.g. to allow `kubelet` to start up even if there is swap is enabled on your server, set this to: `"--fail-swap-on=false"`. Or to specify the node-ip advertised by `kubelet`, set this to `"--node-ip={{ ansible_host }}"`. *This is deprecated. Please use `kubernetes_config_kubelet_configuration` instead.*

    kubernetes_kubeadm_init_extra_opts: ""

Extra args to pass to `kubeadm init` during K8s control plane initialization. E.g. to specify extra Subject Alternative Names for API server certificate, set this to: `"--apiserver-cert-extra-sans my-custom.host"`

    kubernetes_join_command_extra_opts: ""

Extra args to pass to the generated `kubeadm join` command during K8s node initialization. E.g. to ignore certain preflight errors like swap being enabled, set this to: `--ignore-preflight-errors=Swap`

### Additional variables

    kubernetes_allow_pods_on_master: true

Whether to remove the taint that denies pods from being deployed to the Kubernetes master. If you have a single-node cluster, this should definitely be `True`. Otherwise, set to `False` if you want a dedicated Kubernetes master which doesn't run any other pods.

    kubernetes_enable_web_ui: false
    kubernetes_web_ui_manifest_file: https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

Whether to enable the Kubernetes web dashboard UI (only accessible on the master itself, or proxied), and the file containing the web dashboard UI manifest.

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

This role currently supports `flannel` (default), `calico` or `weave` for cluster pod networking. Choose only one for your cluster; converting between them is not done automatically and could result in broken networking; if you need to switch from one to another, it should be done outside of this role.

    kubernetes_apiserver_advertise_address: ''
    kubernetes_version_kubeadm: 'stable-{{ kubernetes_version }}'
    kubernetes_ignore_preflight_errors: 'all'

Options passed to `kubeadm init` when initializing the Kubernetes master. The `kubernetes_apiserver_advertise_address` defaults to `ansible_default_ipv4.address` if it's left empty.

    kubernetes_apt_release_channel: main
    kubernetes_apt_repository: "deb http://apt.kubernetes.io/ kubernetes-xenial {{ kubernetes_apt_release_channel }}"
    kubernetes_apt_ignore_key_error: false

Apt repository options for Kubernetes installation.

    kubernetes_yum_arch: x86_64
    kubernetes_yum_base_url: "https://packages.cloud.google.com/yum/repos/kubernetes-el7-{{ kubernetes_yum_arch }}"
    kubernetes_yum_gpg_key:
      - https://packages.cloud.google.com/yum/doc/yum-key.gpg
      - https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

Yum repository options for Kubernetes installation. You can change `kubernete_yum_gpg_key` to a different url if you are behind a firewall or provide a trustworthy mirror. Usually in combination with changing `kubernetes_yum_base_url` as well.

    kubernetes_flannel_manifest_file_rbac: https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
    kubernetes_flannel_manifest_file: https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

Flannel manifest files to apply to the Kubernetes cluster to enable networking. You can copy your own files to your server and apply them instead, if you need to customize the Flannel networking configuration.

## Dependencies

None.

## Example Playbooks

### Single node (master-only) cluster

```yaml
- hosts: all

  vars:
    kubernetes_allow_pods_on_master: true

  roles:
    - geerlingguy.docker
    - geerlingguy.kubernetes
```

### Two or more nodes (single master) cluster

Master inventory vars:

```yaml
kubernetes_role: "master"
```

Node(s) inventory vars:

```yaml
kubernetes_role: "node"
```

Playbook:

```yaml
- hosts: all

  vars:
    kubernetes_allow_pods_on_master: true

  roles:
    - geerlingguy.docker
    - geerlingguy.kubernetes
```

Then, log into the Kubernetes master, and run `kubectl get nodes` as root, and you should see a list of all the servers.

## License

MIT / BSD

## Author Information

This role was created in 2018 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
