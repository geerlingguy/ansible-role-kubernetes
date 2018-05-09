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

TODO.

    kubernetes_allow_swap: False

TODO.

    kubernetes_allow_pods_on_master: True

TODO.

    kubernetes_enable_web_ui: False

TODO.

    kubernetes_apt_release_channel: main
    kubernetes_apt_repository: "deb http://apt.kubernetes.io/ kubernetes-{{ ansible_distribution_release }} {{ kubernetes_apt_release_channel }}"
    kubernetes_apt_ignore_key_error: False

TODO.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - geerlingguy.docker
    - geerlingguy.kubernetes
```

## License

MIT / BSD

## Author Information

This role was created in 2018 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
