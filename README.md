# Ansible Role: rke2

Install [rke2](https://docs.rke2.io/) on Debian GNU/Linux, together with
[cert-manager](https://cert-manager.io/) and
[local-path-provisioner](https://github.com/rancher/local-path-provisioner),
which provides the default storage class. Ingress is left to RKE2 itself,
the role only adds a NetworkPolicy that lets the bundled controller reach
the backends.

## Requirements
The role is designed for [Debian GNU/Linux](https://debian.org).

## Role Variables

Which release of RKE2 is installed, and which cert-manager and
local-path-provisioner go with it, is configured through these variables.
All of them are optional, the default is the newest release of the `stable`
channel, installed once and never upgraded.

```
# 'stable', 'latest', 'testing' or a minor channel like 'v1.32', see
# https://update.rke2.io/v1-release/channels
rke2_channel: stable
# exact release, takes precedence over the channel, e.g. v1.32.5+rke2r1
rke2_version: v1.32.5+rke2r1
# let the role upgrade an already installed RKE2 to rke2_version
rke2_allow_upgrade: false
# cert-manager release installed into the cluster
rke2_certmanager_version: v1.21.1
# local-path-provisioner release providing the default storage class
rke2_local_path_provisioner_version: v0.0.36
```

On the ILM appliance they are set in the `vars` of `playbooks/ilm.yml`,
which pins `rke2_version` and leaves the rest at the `rke2_default_*` values
listed below. They are deliberately not part of `/etc/ilm-ansible/vars`,
which holds only what the appliance TUI configures.

Every run reports which RKE2 is installed, which one is configured and what
the role is about to do:

```
TASK [rke2 : Report what is going to be done with RKE2] ***
ok: [ilm] => {
    "msg": "installed: v1.35.5+rke2r1; configured: v1.35.6+rke2r1; action: none"
}
```

By default the role installs RKE2 only once, so on later runs the action is
`none` even when the configured version has changed since. With
`rke2_allow_upgrade: true` it upgrades in place instead, following the
[documented procedure](https://docs.rke2.io/upgrades/manual): re-run the
installer, then restart `rke2-server`. That stops and starts the
Kubernetes cluster. Before touching anything the role saves an etcd
snapshot named `pre-upgrade`, and afterwards it waits until the node
reports the new version, so a failed upgrade doesn't pass unnoticed.

Only an exact `rke2_version` can be upgraded to. An `rke2_channel` is
resolved by the installer, so there is nothing to compare the installed
version with, and the role leaves an existing installation alone.

Whether the configured release can be reached from the running one is up to
the operator - the role doesn't check it. Kubernetes can't be downgraded,
and minor releases have to be installed one at a time (e.g. 1.34 -> 1.35 ->
1.36).

### cert-manager and local-path-provisioner

These two need no `rke2_allow_upgrade`: they are upgraded in place whenever
the configured version changes and neither restarts the cluster. Helm
installs or upgrades the cert-manager release to `rke2_certmanager_version`,
and applying the manifest of the configured local-path-provisioner release
updates its Deployment. Both run on every playbook run and are no-ops when
the cluster already matches.

Upgrade compatibility is again the operator's call. cert-manager
[has to be upgraded one minor release at a time](https://cert-manager.io/docs/installation/upgrade/),
always the newest patch of each, and the chart carries its CRDs.
local-path-provisioner is numbered `v0.0.x`, so every change there is a
patch bump.

The local-path-provisioner manifest is downloaded to a file whose name
contains the version. A fixed name can go stale - `get_url` would send
`If-Modified-Since` built from the mtime of the file already there, and a
release tagged earlier than that download answers `304 Not Modified`,
leaving the old manifest in place to be applied again.

### Role level variables

Everything else lives in [defaults/main.yml](defaults/main.yml), so all of it
can be overridden the same way. The `rke2_default_*` ones hold the value each
variable above falls back to when nothing is passed in; they carry their own
names so that a consumer of the role can use them while building its own
value out of its own configuration:

```
rke2_channel: "{{ my_settings.channel | default(rke2_default_channel, true) }}"
```

| variable | default |
|---|---|
| `rke2_default_channel` | `stable` |
| `rke2_default_version` | empty, meaning "resolve the channel" |
| `rke2_default_allow_upgrade` | `false`, set to `true` to upgrade RKE2 in place |
| `rke2_default_certmanager_version` | the cert-manager release to install |
| `rke2_default_local_path_provisioner_version` | the local-path-provisioner release to install |
| `rke2_certmanager_helm_repository_name` | `jetstack` |
| `rke2_certmanager_helm_repository_url` | `https://charts.jetstack.io`, point it elsewhere to install the chart from a local mirror |
| `rke2_install_local_path_provisioner` | `true`, set to `false` to leave the cluster without a default storage class |
| `rke2_base_kube_cfg_dir` | list of kubeconfig directories (default: /root/.kube owned by root) |

`custom_kube_cfg_dir` is optional and takes the same shape as
`rke2_base_kube_cfg_dir`: the kubeconfig is copied to `/root/.kube` plus
every directory listed there, see the example playbook below.

If you have to use HTTP_PROXY to access Internet, please visit [ansible role http_proxy](https://github.com/semik/ansible-role-http-proxy/tree/split#role-variables) for info howto provide the role with info about the Proxy.

## Example Playbook
```
- hosts: localhost
  connection: local
  vars:
    custom_kube_cfg_dir:
      - owner: "semik"
        group: "semik"
        dir: "/home/semik/.kube"

  roles:
    - role: rke2
```
