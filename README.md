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
  local_path_provisioner_version: v0.0.29
```

By default the role installs RKE2 only once and only warns when an exact
`version` is configured and a different one is already installed. With
`allow_upgrade: true` it upgrades in place instead, following the
[documented procedure](https://docs.rke2.io/upgrades/manual): re-run the
installer, then restart `rke2-server`. That stops and starts the
Kubernetes cluster. Before touching anything the role saves an etcd
snapshot named `pre-upgrade-<from>-to-<to>`, and afterwards it waits
until the node reports the new version, so a failed upgrade doesn't pass
unnoticed.

An upgrade is refused, with the reason in the warning, when:

* only a `channel` is configured - it is resolved by the installer, so
  there is nothing to compare the installed version with,
* it would be a downgrade, or
* it would skip a Kubernetes minor release, e.g. 1.34 to 1.36. Those have
  to be installed one at a time.

### cert-manager and local-path-provisioner

These two are handled differently from RKE2, because they are upgraded in
place on their own whenever the configured version changes and neither
restarts the cluster - Helm upgrades the cert-manager release, and applying
the manifest of a new local-path-provisioner release updates its
Deployment. They therefore need no `allow_upgrade`, changing the value
upgrades the component on the next run.

Two moves are still refused, and in that case the task which would apply
the change is skipped, so the running component is left untouched:

* a downgrade, and
* a jump over a minor release. cert-manager
  [doesn't support it](https://cert-manager.io/docs/installation/upgrade/) -
  minor releases have to be installed one at a time, always the newest
  patch of each. local-path-provisioner is numbered `v0.0.x`, so every
  upgrade is a patch bump there and only a downgrade is ever refused.

The installed version is read from the cluster: from the Helm release for
cert-manager, and from the tag of the Deployment's image for
local-path-provisioner.

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
| `rke2_fail_on_version_mismatch` | `false`, set to `true` to make a refused version change fatal |

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
