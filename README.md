# Ansible Role: rke2

Install [rke2|https://docs.rke2.io/] + ingress-nginx and certmanager on Debian GNU/Linux.

## Requirements
The role is designed for [Debian GNU/Linux](https://debian.org).

## Role Variables

```
ingress:
  certificate_file: certificate-for-https-server
  private_key_file: private-key-for-https-server
```

If you have to use HTTP_PROXY to access Internet, please visit [ansible role http_proxy](https://github.com/semik/ansible-role-http-proxy/tree/split#role-variables) for info howto provide the role with info about the Proxy.

## Example Playbook
```
- hosts: localhost
  connection: local

  roles:
    - role: rke2
```
