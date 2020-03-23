# Ansible Role: Kubernetes

[![Build Status](https://travis-ci.org/geerlingguy/ansible-role-kubernetes.svg?branch=master)](https://travis-ci.org/geerlingguy/ansible-role-kubernetes)

An Ansible Role that installs [Kubernetes](https://kubernetes.io) on Linux.

## Requirements

Requires Docker; recommended role for Docker installation: `geerlingguy.docker`.

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

    kubernetes_version: '1.17'
    kubernetes_version_rhel_package: '1.17.2'

The minor version of Kubernetes to install. The plain `kubernetes_version` is used to pin an apt package version on Debian, and as the Kubernetes version passed into the `kubeadm init` command (see `kubernetes_version_kubeadm`). The `kubernetes_version_rhel_package` variable must be a specific Kubernetes release, and is used to pin the version on Red Hat / CentOS servers.

    kubernetes_role: master

Whether the particular server will serve as a Kubernetes `master` (default) or `node`. The master will have `kubeadm init` run on it to intialize the entire K8s control plane, while `node`s will have `kubeadm join` run on them to join them to the `master`.

    kubernetes_kubelet_extra_args: ""
    kubernetes_kubelet_extra_args_config_file: /etc/default/kubelet

Extra args to pass to `kubelet` during startup. E.g. to allow `kubelet` to start up even if there is swap is enabled on your server, set this to: `"--fail-swap-on=false"`. Or to specify the node-ip advertised by `kubelet`, set this to `"--node-ip={{ ansible_host }}"`.

    kubernetes_kubeadm_init_extra_opts: ""

Extra args to pass to `kubeadm init` during K8s control plane initialization. E.g. to specify extra Subject Alternative Names for API server certificate, set this to: `"--apiserver-cert-extra-sans my-custom.host"`

    kubernetes_join_command_extra_opts: ""

Extra args to pass to the generated `kubeadm join` command during K8s node initialization. E.g. to ignore certain preflight errors like swap being enabled, set this to: `--ignore-preflight-errors=Swap`

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

Yum repository options for Kubernetes installation.

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
