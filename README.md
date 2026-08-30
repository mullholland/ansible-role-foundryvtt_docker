# [Ansible role foundryvtt_docker](#foundryvtt_docker)

Installs and configures foundryvtt container based on felddy/foundryvtt-docker

|GitHub|Downloads|Version|
|------|---------|-------|
|[![github](https://github.com/mullholland/ansible-role-foundryvtt_docker/actions/workflows/molecule.yml/badge.svg)](https://github.com/mullholland/ansible-role-foundryvtt_docker/actions/workflows/molecule.yml)|[![downloads](https://img.shields.io/ansible/role/d/mullholland/foundryvtt_docker)](https://galaxy.ansible.com/mullholland/foundryvtt_docker)|[![Version](https://img.shields.io/github/release/mullholland/ansible-role-foundryvtt_docker.svg)](https://github.com/mullholland/ansible-role-foundryvtt_docker/releases/)|
## [Example Playbook](#example-playbook)

This example is taken from [`molecule/default/converge.yml`](https://github.com/mullholland/ansible-role-foundryvtt_docker/blob/master/molecule/default/converge.yml) and is tested on each push, pull request and release.

```yaml
---
- name: Converge
  hosts: all
  gather_facts: true
  vars:
    foundryvtt_docker_secrets_enable: true
    foundryvtt_docker_secrets:
      - 'foundry_admin_key: "SuperSecret"'

  roles:
    - role: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}"
```

The machine needs to be prepared. In CI this is done using [`molecule/default/prepare.yml`](https://github.com/mullholland/ansible-role-foundryvtt_docker/blob/master/molecule/default/prepare.yml):

```yaml
---
- name: Prepare
  hosts: all
  gather_facts: true
  vars:
    pip_packages:
      - "docker>=7.1.0"

  roles:
    - role: mullholland.docker
    - role: mullholland.repository_epel
    - role: mullholland.pip

  tasks:
    - name: Install python3-docker on Debian 12+ / Ubuntu 24.04+
      ansible.builtin.apt:
        name: python3-docker
        state: present
      when: (ansible_distribution == "Debian" and ansible_distribution_major_version | int >= 12) or
            (ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 24)

    # mullholland.pip only installs the pip binary itself, it no longer
    # installs pip packages (removed due to PEP 668), so the
    # community.docker modules used below need their Python dependency
    # installed here directly on every other distribution.
    - name: Install python dependencies for community.docker modules
      ansible.builtin.pip:
        name: "{{ pip_packages }}"
        state: present
      when: not ((ansible_distribution == "Debian" and ansible_distribution_major_version | int >= 12) or
                 (ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 24))

    # Nested overlay2-on-overlay2 mounts can fail on some container backends
    # (e.g. OrbStack, some CI runners): "failed to mount ...: fstype: overlay
    # ... invalid argument". vfs avoids that at the cost of slower image
    # pulls, which is an acceptable trade-off for a molecule test instance.
    - name: Configure inner dockerd to use the vfs storage driver
      ansible.builtin.copy:
        dest: /etc/docker/daemon.json
        content: |
          {
            "storage-driver": "vfs"
          }
        mode: "0644"
      notify: restart docker

  handlers:
    - name: restart docker
      ansible.builtin.systemd:
        name: docker
        state: restarted
```


## [Role Variables](#role-variables)

The default values for the variables are set in [`defaults/main.yml`](https://github.com/mullholland/ansible-role-foundryvtt_docker/blob/master/defaults/main.yml):

```yaml
---
# IMPORTANT
# You should configure the FOUNDRY_USERNAME and FOUNDRY_PASSWORD
# either via secrets or environment varaibles to get a running foundryvtt container
# without it, the container will start with an error

# General config
foundryvtt_docker_network_name: "web"
foundryvtt_docker_base_path: "/opt"
foundryvtt_docker_timezone: "Europe/Berlin"

# User/Group of the stack. Everything is mapped to this, instead of root.
foundryvtt_docker_user: "homelab"
foundryvtt_docker_uid: "900"
foundryvtt_docker_group: "homelab"
foundryvtt_docker_gid: "900"
foundryvtt_docker_user_system: true

# which container version to install
# https://github.com/felddy/foundryvtt-docker
# can also be latest
foundryvtt_docker_version: "felddy/foundryvtt:latest"

# Secrets
foundryvtt_docker_secrets_enable: false

# secrets are saved in the config.json and added via [docker secrets](https://docs.docker.com/engine/swarm/secrets/)
# Will be ignored if empty
foundryvtt_docker_secrets: []
# Overrides FOUNDRY_ADMIN_KEY environment variable.
# - '"foundry_admin_key": ""'
# Overrides FOUNDRY_LICENSE_KEY environment variable.
# - '"foundry_license_key": ""'
# Overrides FOUNDRY_PASSWORD environment variable.
# - '"foundry_password": ""'
# Overrides FOUNDRY_PASSWORD_SALT environment variable.
# - '"foundry_password_salt": ""'
# Overrides FOUNDRY_USERNAME environment variable.
# - '"foundry_username": ""'

# additional docker compose environment variables
# https://github.com/felddy/foundryvtt-docker?tab=readme-ov-file#optional-variables
foundryvtt_docker_environment_variables:
  - 'FOUNDRY_HOSTNAME: "{{ inventory_hostname }}"'
  - 'FOUNDRY_IP_DISCOVERY: false'
  - 'FOUNDRY_MINIFY_STATIC_FILES: true'
# which port to expose. can be empty if used with traefik for example
foundryvtt_docker_port: "30000"
foundryvtt_docker_labels:
  - "traefik.enable=true"
  - "traefik.http.routers.foundry.entryPoints=https"
  - "traefik.http.routers.foundry.rule=Host(`foundry.example.com`)"
  - "traefik.http.routers.foundry.tls.certResolver=letsEncrypt"
  - "traefik.http.routers.foundry.tls=true"
```

## [Requirements](#requirements)

- pip packages listed in [requirements.txt](https://github.com/mullholland/ansible-role-foundryvtt_docker/blob/master/requirements.txt).

## [State of used roles](#state-of-used-roles)

The following roles are used to prepare a system. You can prepare your system in another way.

| Requirement | GitHub | GitLab |
|-------------|--------|--------|
|[mullholland.repository_epel](https://galaxy.ansible.com/mullholland/repository_epel)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-repository_epel/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-repository_epel/actions)|[![Build Status GitLab](https://gitlab.com/mullholland-github-mirror/ansible-role-repository_epel/badges/master/pipeline.svg)](https://gitlab.com/mullholland-github-mirror/ansible-role-repository_epel)|
|[mullholland.docker](https://galaxy.ansible.com/mullholland/docker)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-docker/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-docker/actions)|[![Build Status GitLab](https://gitlab.com/mullholland-github-mirror/ansible-role-docker/badges/master/pipeline.svg)](https://gitlab.com/mullholland-github-mirror/ansible-role-docker)|
|[mullholland.pip](https://galaxy.ansible.com/mullholland/pip)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-pip/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-pip/actions)|[![Build Status GitLab](https://gitlab.com/mullholland-github-mirror/ansible-role-pip/badges/master/pipeline.svg)](https://gitlab.com/mullholland-github-mirror/ansible-role-pip)|

## [Context](#context)

This role is a part of many compatible roles. Have a look at [the documentation of these roles](https://mullholland.net) for further information.

## [Compatibility](#compatibility)

This role has been tested on these [container images](https://hub.docker.com/u/mullholland):

|container|tags|
|---------|----|
|[EL](https://hub.docker.com/r/mullholland/enterpriselinux)|all|
|[Fedora](https://hub.docker.com/r/mullholland/fedora/)|all|
|[Rocky](https://hub.docker.com/r/mullholland/rockylinux)|all|
|[AlmaLinux](https://hub.docker.com/r/mullholland/almalinux)|all|
|[Ubuntu](https://hub.docker.com/r/mullholland/ubuntu)|all|
|[Debian](https://hub.docker.com/r/mullholland/debian)|all|
|[CentOS](https://hub.docker.com/r/mullholland/centos)|all|

The minimum version of Ansible required is 2.10, tests have been done to:

- The version before the previous version.
- The previous version.
- The current version.

If you find issues, please register them in [GitHub](https://github.com/mullholland/ansible-role-foundryvtt_docker/issues).

## [License](#license)

[MIT](https://github.com/mullholland/ansible-role-foundryvtt_docker/blob/master/LICENSE).

## [Author Information](#author-information)

[mullholland](https://mullholland.net)
