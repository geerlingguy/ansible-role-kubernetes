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
      - name: kubeadm
        state: present
      - name: kubernetes-cni
        state: present

Kubernetes packages to be installed on the server. You can either provide a list of package names, or set `name` and `state` to have more control over whether the package is `present`, `absent`, `latest`, etc.

    kubernetes_kubelet_extra_args: ""

Extra args to pass to `kubelet` during startup. E.g. to allow `kubelet` to start up even if there is swap is enabled on your server, set this to: `"--fail-swap-on=false"`.

    kubernetes_allow_pods_on_master: True

Whether to remove the taint that denies pods from being deployed to the Kubernetes master. If you have a single-node cluster, this should definitely be `True`. Otherwise, set to `False` if you want a dedicated Kubernetes master which doesn't run any other pods.

    kubernetes_enable_web_ui: False

Whether to enable the Kubernetes web dashboard UI (only accessible on the master itself, or proxied).

    kubernetes_pod_network_cidr: '10.0.1.0/16'
    kubernetes_version: 'stable-1.10'
    kubernetes_ignore_preflight_errors: 'all'

Options passed to `kubeadm init` when initializing the Kubernetes master.

    kubernetes_apt_release_channel: main
    kubernetes_apt_repository: "deb http://apt.kubernetes.io/ kubernetes-xenial {{ kubernetes_apt_release_channel }}"
    kubernetes_apt_ignore_key_error: False

Apt repository options for Kubernetes installation.

    kubernetes_yum_arch: x86_64

Yum repository options for Kubernetes installation.

## Dependencies

None.

## Example Playbook

For a single node (master) Kubernetes cluster:

```yaml
- hosts: all

  vars:
    kubernetes_allow_pods_on_master: True

  roles:
    - geerlingguy.docker
    - geerlingguy.kubernetes
```

## License

MIT / BSD

## Author Information

This role was created in 2018 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
