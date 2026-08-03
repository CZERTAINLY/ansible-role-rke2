# Ansible Role: rke2

Install [rke2](https://docs.rke2.io/) + ingress-nginx and certmanager on Debian GNU/Linux.

## Requirements
The role is designed for [Debian GNU/Linux](https://debian.org).

## Role Variables

```
ingress:
  certificate_file: certificate-for-https-server
  private_key_file: private-key-for-https-server
```

Which release of RKE2 is installed. All keys are optional, the default is
the newest release of the `stable` channel, installed once and never
upgraded.

```
rke2:
  # 'stable', 'latest', 'testing' or a minor channel like 'v1.32', see
  # https://update.rke2.io/v1-release/channels
  channel: stable
  # exact release, takes precedence over channel, e.g. v1.32.5+rke2r1
  version: v1.32.5+rke2r1
  # let the role upgrade an already installed RKE2 to 'version'
  allow_upgrade: false
  # cert-manager release installed into the cluster
  certmanager_version: v1.21.1
  # local-path-provisioner release providing the default storage class
  local_path_provisioner_version: v0.0.36
```

By default the role installs RKE2 only once, and on later runs it only
reports that the installed version differs from the configured one. With
`allow_upgrade: true` it upgrades in place instead, following the
[documented procedure](https://docs.rke2.io/upgrades/manual): re-run the
installer, then restart `rke2-server`. That stops and starts the
Kubernetes cluster. Before touching anything the role saves an etcd
snapshot named `pre-upgrade`, and afterwards it waits until the node
reports the new version, so a failed upgrade doesn't pass unnoticed.

Only an exact `version` can be upgraded to. A `channel` is resolved by the
installer, so there is nothing to compare the installed version with, and
the role leaves an existing installation alone.

Whether the configured release can be reached from the running one is up to
the operator - the role doesn't check it. Kubernetes can't be downgraded,
and minor releases have to be installed one at a time (e.g. 1.34 → 1.35 →
1.36).

### cert-manager and local-path-provisioner

These two need no `allow_upgrade`: they are upgraded in place whenever the
configured version changes and neither restarts the cluster. Helm installs
or upgrades the cert-manager release to `certmanager_version`, and applying
the manifest of the configured local-path-provisioner release updates its
Deployment. Both run on every playbook run and are no-ops when the cluster
already matches.

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

Role level fallbacks, used when the `rke2` dict isn't defined at all:

| variable | default |
|---|---|
| `rke2_default_channel` | `stable` |
| `rke2_default_version` | empty, meaning "resolve the channel" |
| `rke2_allow_upgrade` | `false`, set to `true` to upgrade RKE2 in place |
| `rke2_default_certmanager_version` | the cert-manager release to install |
| `rke2_default_local_path_provisioner_version` | the local-path-provisioner release to install |

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
