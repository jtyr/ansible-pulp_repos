pulp_repos
==========

Ansible role which helps to manage Pulp repos in semi-automated way.

---

Repositories created by this role have a name in the following format:

```
<group>-<platform><version>-<name>-<tag>
```

where each of the elements is:

- `<group>` - Group of repositories preventing conflicts between the names of
  internal and external repositories (e.g. internal project called `graphite`
  and external project called `graphite`). The groups are:
    - `base` - OS-related repositories (e.g. `updates`, `extras`, ...).
    - `external` - External repositories (e.g. `postgresql`, `pulp`, ...).
    - `misc` - Repositories with packages which don't have external YUM
      repository.
    - `release` - Repositories for the internal projects.
- `<platform>` - On which platform the package was built (e.g. `el`, `centos`,
  ...).
- `<version>` - Platform version (e.g. `6`, `22`, ...). Can be omitted if the
  RPMs can be installed to all platform versions.
- `<name>` - Name of the project (e.g. `epel`, `postgresql`, `extras`, ...)
- `<tag>` - Tag specifying version of the repository (e.g. `1`, `201605_1`,
  `2016Q1_1`, `20160518_1`, ...).

Examples of repository names:

```
base-el6-extras-201605_1
base-el6-updates-201604_3
release-el6-myapp-1
release-el6-otherapp-1
misc-el6-oraclejava-1
misc-el6-infra-1
external-el6-pulp-201605_2
external-el6-postgresql-201605_1
```

The repository name corresponds to the relative directory structure from which
the packages are served by Pulp. The directory structure has the following
format:

```
<group>/<platform>/<version>/<name>/<tag>
```

where all the elements have the same meaning and value as in the case of the
repository name. If `<version>` is omitted in the repository name, keyword
`all` is used in the directory structure. Examples of directory structures:

```
https://pulp.server.local/pulp/repos/base/el/6/extras/201605_1
https://pulp.server.local/pulp/repos/base/el/6/updates/201604_3
https://pulp.server.local/pulp/repos/release/el/6/myapp/1
https://pulp.server.local/pulp/repos/release/el/6/otherapp/1
https://pulp.server.local/pulp/repos/misc/el/6/oraclejava/1
https://pulp.server.local/pulp/repos/misc/el/6/infra/1
https://pulp.server.local/pulp/repos/external/el/6/pulp/201605_2
https://pulp.server.local/pulp/repos/external/el/6/postgresql/201605_1
```

There is no need to indicate the architecture of the platform (e.g. `i686`,
`x86_64`, ...) in the repository name nor in the directory structure as most
people is using `x86_64` only. That keeps the repository name resp. the
directory structure reasonably short resp. shallow.


Examples
--------

```
---

- name: Example of how to create and delete repositories
  hosts: myhost
  vars:
    # Additional certs used for fetching packages from some repos
    pulp_repos__certs: &pulp_repos__certs
      ca_cert: "{{ lookup('file', 'ca.crt') }}"
      client_key: "{{ lookup('file', 'client.crt') }}"
      client_cert: "{{ lookup('file', 'client.pem') }}"
    # Create or delete repos
    pulp_repos_create:
      # Create a repo with a feed
      - name: epel
        feed: https://dl.fedoraproject.org/pub/epel/6/x86_64/
        gpgkey: https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-6
        group: external
        platform: el
        platform_version: 6
        tag: "201605_1"
      # Create a repo with a feed and additional certificates
      - <<: *pulp_repos__certs
        name: scalablefilesystem
        feed: https://rhua.redhat.com/pulp/repos/content/dist/rhel/rhui/server/6/6Server/x86_64/scalablefilesystem/os
        description: Red Hat Enterprise Linux Scalable File System (for RHEL 6 Server) (RPMs) from RHUI
        group: base
        platform: el
        platform_version: 6
        tag: "201605_1"
      # Create a repo with no feed
      - name: oraclejava
        group: misc
        platform: el
        platform_version: 6
        tag: 1
      # Delete a repo
      - name: scalablefilesystem
        group: base
        platform: el
        platform_version: 6
        tag: "201604_1"
        stete: absent
  roles:
    - role: pulp_repos
      tags: pulp_repos
```

Using this role, it's possible to target creation of only a specific repository
by specifying the `pulp_repos_limit` parameter as an extra variable on the
command line:

```
# This will target only the oraclejava and the epel repository (the
# scalablefilesystem will be skipped):
$ ansible-playbook -i hosts -l pulp.server.local -t pulp_repos -e '{ pulp_repos_limit: [ misc-el6-oraclejava-1, external-el6-epel-201605_1 ] }' ./site.yaml
```


Role variables
--------------

List of variables used by the role:

```
# Lists of repositories to be created
pulp_repos_create: []

# Pulp authentication details
pulp_repos_auth_user: admin
pulp_repos_auth_password: admin

# Validate Pulp server SSL certs
pulp_repos_validate_certs: no

# Auto-publish after repo sync
pulp_repos_distributor_auto_publish: yes

# Publish over HTTP/HTTPS
pulp_repos_distributor_publish_http: yes
pulp_repos_distributor_publish_https: yes

# Apache rewrite rule for RHEL Server
pulp_repos_apache_rewrite_config:
  content:
    - sections:
      - name: Location
        param: /pulp/repos/
        content:
          - options:
            - RewriteEngine: "on"
            - RedirectMatch:
              - ^(/pulp/repos/(base|external|misc|release)/[^/\.]+/\d+)Server(/.*)$
              - $1$3

# Data used to create the Pulp repo
# (it must be as a variable because we need to translate it to JSON in order we
# get the 'null' string translated to real null)
pulp_repos_create_data:
  id: "{{ repo.group }}-{{ repo.platform }}{{ repo.platform_version | default('') }}-{{ repo.name }}-{{ repo.tag }}"
  description: "{{ repo.description | default('null') }}"
  notes:
    _repo-type: rpm-repo
  importer_type_id: yum_importer
  importer_config:
    feed: "{{ repo.feed | default('null') }}"
    proxy_host: "{{ repo.proxy_host | default('null') }}"
    proxy_port: "{{ repo.proxy_port | default('null') }}"
    ssl_ca_cert: "{{ repo.ca_cert | default('null') }}"
    ssl_client_key: "{{ repo.client_key| default('null') }}"
    ssl_client_cert: "{{ repo.client_cert | default('null') }}"
  distributors:
    - distributor_type_id: yum_distributor
      distributor_id: yum_distributor
      auto_publish: "{{ pulp_repos_distributor_auto_publish }}"
      distributor_config:
        http: "{{ pulp_repos_distributor_publish_http }}"
        https: "{{ pulp_repos_distributor_publish_https }}"
        relative_url: "{{ repo.group }}/{{ repo.platform }}/{{ repo.platform_version | default('all') }}/{{ repo.name }}/{{ repo.tag }}"
    - distributor_type_id: export_distributor
      distributor_id: export_distributor
      auto_publish: "{{ pulp_repos_distributor_auto_publish }}"
      distributor_config:
        http: "{{ pulp_repos_distributor_publish_http }}"
        https: "{{ pulp_repos_distributor_publish_https }}"
        relative_url: "{{ repo.group }}/{{ repo.platform }}/{{ repo.platform_version | default('all') }}/{{ repo.name }}/{{ repo.tag }}"
```


Dependencies
------------

- [`config_encoder_filters`](https://github.com/jtyr/ansible-config_encoder_filters)
- [`pulp`](https://github.com/jtyr/ansible-pulp) role (optional)


License
-------

MIT


Author
------

Jiri Tyr
