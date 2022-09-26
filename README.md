# Ansible meta-role: mergevars

[![CI](https://github.com/ahosie/ansible-role-mergevars/workflows/CI/badge.svg?event=push)](https://github.com/ahosie/ansible-role-mergevars/actions?query=workflow%3ACI)


## Description

Some ansible roles make use of hierarchical variables, i.e. those where the values are defined within a nested hashmap/dictionary/&lt;insert equivalent structure here&gt;, or require a single instance of a list be defined.

Such roles can be unwieldy to configure if variables need to be defined on both a host and within one or more groups.


This ansible role attempts to work around this by allowing a developer to define a **variable pattern** that will be flattened down into a single variable.


## Role Variables


### mergevars_lists
To merge multiple lists into a single variable - define the `mergevars_lists` as follows: 

```yaml
mergevars_lists:
  - variable: foo
    pattern: 'bar_.+'
```

Any variables that start with the name `bar_` (when this role is invoked) will be merged together in a single list named `foo`.


### mergevars_dicts
To merge several dictionaries - define the `mergevars_dicts` as follows:
```yaml
mergevars_dicts:
  - variable: foo
    pattern: 'bar_.+'
```

Any variables that start with the name `bar_` (when this role is invoked) will be merged together in a single list named `foo`.

e.g. in a vars file (`vars/demo.yml`) you could define:
```yaml
bar_the_first:
  name: 'Foo McFoo Face'
  purpose: 'Demo'
bar_the_second:
  description: 'Demonstrate the thing'
```

If a play (`demo.yml`) contained the following statements:
```yaml
- hosts: all
  connection: local
  vars_files:
    - vars/demo.yml
  vars:
    mergevars_dicts:
      - variable: foo
        pattern: 'bar_.+'
  roles:
    - ahosie.mergevars
  tasks:
    - name: "Display the merged variable foo containing all bar_'s"
      debug:
        msg: "{{ foo }}"
```

Then the result of running `ansible-play demo.yml` would look like:
```text
TASK [Display the merged variable foo containing all bar_'s] *****************************************************************************************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "description": "Demonstrate the thing",
        "name": "Foo McFoo Face",
        "purpose": "Demo"
    }
}

PLAY RECAP *******************************************************************************************************************************************************************************************************************************************************************************************
localhost                  : ok=7    changed=3    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```


## Example Usage


### New Relic Logging

The New Relic Infrastructure Agent ansible playbook requires a single `nrinfragent_logging` variable be defined that references each of the log files/facilities/streams that will be forwarded to New Relic.

You could define all the log sources as a monolithic host or group variable - or by including this role you can declare various log sources in appropriate groups and merge together at runtime.

`hosts.yml`
```yaml
all:
  children:
    debian:
    rhel:
    linux:
linux:
  children:
    rhel:
    debian:
debian:
  hosts:
    router:
    loadbalancer:
rhel:
  hosts:
    webserver1:
    webserver2:
    webserver3:
```

`deploy_newrelic_logging.yml`
```yaml
---
- hosts: all
  vars:
    mergevars_lists:
      - variable: 'nrinfragent_logging'
        pattern: 'newrelic_logs_.+'
  roles:
    - ahosie.mergevars
    - newrelic.newrelic-infra
```

```group_vars/debian.yml```
```yaml
newrelic_logs_debian:
  - name: packages
    file: /var/log/dpkg.log
  - name: syslog
    file: /var/log/syslog
    attributes:
      logtype: linux_messages
  - name: auth
    file: /var/log/auth
```

```group_vars/rhel.yml```
```yaml
newrelic_logs_rhel:
  - name: packages
    file: /var/log/yum.log
  - name: syslog
    file: /var/log/messages
    attributes:
      logtype: linux_messages
  - name: audit
    file: /var/log/audit
```

```group_vars/linux.yml```
```yaml
newrelic_logs_linux:
  - name: dmesg
    file: /var/log/dmesg
  - name: boot
    file: /var/log/boot.log
  - name: xorg
    file: /var/log/Xorg.0.log
```
