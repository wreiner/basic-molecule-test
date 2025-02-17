# Testing Ansible Role With Molecule

Very basic example on how to test Ansible roles with Molecule.

## Basic Setup

- Install ansible and molecule

```
python3 -m venv ansiblevenv
source ansiblevenv/bin/activate
pip install molecule molecule-plugins[docker|podman] ansible pytest testinfra
```

- Create a role from templates

```
ansible-galaxy role init nginx
```

- Initialize molecule

```
molecule init scenario --driver-name docker|podman
```

- Update information in `nginx/meta/main.yml`

```
---
galaxy_info:
  role_name: nginx
  namespace: wrt
...
```

- Rename unneeded files from the molecule init step

```
mv -f nginx/molecule/default/create.yml nginx/molecule/default/ocreate.yml
mv -f nginx/molecule/default/destroy.yml nginx/molecule/default/odestroy.yml
```

## Implement role and configure Molecule

- Implement role

- Configure Molecule

```
---
driver:
  name: podman

platforms:
  - name: ubi9-container
    hostname: ubi9-container
    image: registry.access.redhat.com/ubi9/ubi
    privileged: true
    command: "/sbin/init"

provisioner:
  inventory:
    hosts:
      all:
        hosts:
          ubi9-container:
            ansible_python_interpreter: /usr/bin/python3
  name: ansible

verifier:
  name: testinfra
```

- Implement converge playbook, it installs the role

```
$ cat nginx/molecule/default/converge.yml
---
- name: Verify Install Nginx Role
  hosts: all
  roles:
    - nginx
```

- Implement the tests

```
$ cat nginx/molecule/default/tests/test_default.py
import os

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts('all')

# def test_user(host):
#     user = host.user("www-data")
#     assert user.exists

def test_nginx_is_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed

def test_nginx_running_and_enabled(host):
    nginx1 = host.service("nginx")
    assert nginx1.is_running
    assert nginx1.is_enabled

def test_nginx_is_listening(host):
    assert host.socket('tcp://127.0.0.1:80').is_listening
```

All `.py` files in `nginx/molecule/default/tests` will be run as tests.

### Run commands inside the container to prepare it for the test (optional)

```
$ cat nginx/molecule/default/molecule.yml
...
provisioner:
  ...
  name: ansible
  log: true
  playbooks:
    prepare: ../prepare.yml
```

This will run the playbook in the prepare.yml:

```
---
- name: Prepare test container
  hosts: all
  tasks:
    - name: Install dependencies for testing
      ansible.builtin.package:
        name:
          - python3-pip
          - python3
        state: present

    - name: Install pytest and testinfra
      ansible.builtin.pip:
        name:
          - pytest
          - testinfra
```

## Molecule stages

- Run all stages and test the role

```
molecule test
```

- Create the environment

```
molecule create
```

This will create a podman container.

```
(ansvenv) [wreiner@localhost nginx_role]$ podman ps
CONTAINER ID  IMAGE                                                                COMMAND     CREATED         STATUS         PORTS       NAMES
1b4fd01a99b4  localhost/molecule_local/registry.access.redhat.com/ubi9/ubi:latest  /sbin/init  31 seconds ago  Up 31 seconds              ubi9-container
```

- Create the environment and run the role

```
molecule converge
```

The converge playbook will be run a second time to check for idempotency.

- Perform the actual tests

```
molecule verify
```

- Destroy the environment

```
molecule destroy
```

This will stop and delete the podman container.

```
(ansvenv) [wreiner@localhost nginx_role]$ podman ps
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

- Login to testing environment

```
molecule login
```

Or use podman|docker to exec into the container.

## Linting

You should run linting. It was removed from molecule and should be run seperately in CI/CD.
- [https://github.com/ansible/molecule/discussions/3914]
- [https://github.com/ansible/molecule/pull/3802]
