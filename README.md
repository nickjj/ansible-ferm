## What is ansible-ferm? [![Build Status](https://secure.travis-ci.org/nickjj/ansible-ferm.png)](http://travis-ci.org/nickjj/ansible-ferm)

It is an [ansible](http://www.ansible.com/home) role to manage iptables using the ever so flexible ferm tool.

### What problem does it solve and why is it useful?

Working with iptables directly can be really painful and the ufw module is decent for basic needs but sometimes you need a bit more control. I also like the approach of writing templates rather than executing allow/deny commands with ufw. I feel like it sets the tone for a more idempotent setup.

This role is basically copy/pasted from the [ginas project]https://github.com/ginas/ginas/tree/master/playbooks/roles/ginas.ferm. Ginas is one of the best ansible examples I've found, it was created by [@drybjed](https://twitter.com/drybjed).

## Role variables

Below is a list of default values along with a description of what they do.

```
# Should the firewall be enabled?
ferm_enabled: true

# Should ferm do ip-based tagging/locking when it detects someone is trying to port scan you?
ferm_limit_portscans: false

# The default actions to take for certain policies. You likely want to keep these at the default values. This ensures all ports are blocked until you white list them.
ferm_default_policy_input: DROP
ferm_default_policy_output: ACCEPT
ferm_default_policy_forward: DROP

# The lists to use to provide your own rules. This is explained more below.
ferm_input_list: []
ferm_input_group_list: []
ferm_input_host_list: []

# The amount in seconds to cache apt-update.
apt_cache_valid_time: 86400
```

### `ferm_input_list` with the `dport_accept` template

The use case for this would be to white list ports to be opened.

```
  - role: nickjj.ferm
    tags: ferm
    ferm_input_list:
        # Choose the template to use.
        # REQUIRED: It can be either `dport_accept` or `dport_limit`.
      - type: "dport_accept"

        # Which protocol should be used?
        # OPTIONAL: Defaults to tcp.
        protocol: "tcp"

        # Which ports should be open?
        # REQUIRED: It can be the port value or a service in `/etc/services`.
        dport: ["http", "https"]

        # Which IP addresses should be white listed?
        # OPTIONAL: Defaults to an empty list.
        item.saddr: []

        # Should all IP addresses be white listed?
        # OPTIONAL: Defaults to true.
        accept_any: true

        # Which filename should be written out?
        # OPTIONAL: Defaults to the first port listed in `dport`.

        # The filename which will get written to `/etc/ferm/filter-input.d/nginx_accept`.
        filename: "nginx_accept"
```

### `ferm_input_list` with the `dport_limit` template

The use case for this would be to limit connections on specific ports based on an amount of time. This could be used to harden your security.

```
  - role: nickjj.ferm
    tags: ferm
    ferm_input_list:
        # Choose the template to use.
        # REQUIRED: It can be either `dport_accept` or `dport_limit`.
      - type: "dport_limit"

        # Which protocol should be used?
        # OPTIONAL: Defaults to tcp.
        protocol: "tcp"

        # Which ports should be open?
        # REQUIRED: It can be the port value or a service in `/etc/services`.
        dport: ["ssh"]

        # How many seconds to count in between the hits?
        # OPTIONAL: Defaults to 300.
        seconds: "300"

        # How many connections should be allowed per the amount of seconds you specify.
        # OPTIONAL: Defaults to 5.
        hits: "5"

        # Should this rule be disabled?
        # OPTIONAL: Defaults to false.
        disabled: false
```

### `ferm_input_group_list` / `ferm_input_host_list` with the either template

This would be the same as above except it would be scoped to the groups and hosts list.

## Common plays in your playbook

This role expects an `ansible_controller` fact to be set. Notice how the play is ran without sudo. This is necessary for ansible to populate `ansible_env.SSH_CLIENT`.

What this does is set a rule so that only the computer running the ansible playbook will be able to connect on the ssh port. It is dynamic and completely hands free because if you want to run the playbook from a different machine in the future it will automatically update the IP address.

```
---
- name: ensure all servers are commonly configured (without sudo)
  hosts: all
  sudo: false

  tasks:
    - name: ensure IP address of the ansible controller is set
      set_fact:
        ansible_controller: '{{ ansible_env.SSH_CLIENT.split(" ") | first }}/32'
      when: (ansible_controller is undefined and ansible_connection != "local")
      tags: ferm

- name: ensure all servers are commonly configured (with sudo)
  hosts: all
  sudo: true

  roles:
    - { role: nickjj.ferm, tags: [common, ferm] }
```

## Example app play in your playbook

For the sake of this example let's assume you have a group called **app** and you have a typical `site.yml` file.

To use this role edit your `site.yml` file to look something like this:

```
---
- name: ensure app servers are configured
- hosts: app

  roles:
    - role: nickjj.ferm
      tags: [app, ferm]
      ferm_input_list:
        - type: "dport_accept"
          dport: ["http", "https"]
          filename: "nginx_accept"
```

The above assumes you would be using nginx to reverse proxy your app but that isn't necessary. I only chose the `nginx_accept` filename because I use nginx. You can name it whatever you want or even remove the filename to have this role automatically generate a filename for you.

This file will be written to `/etc/ferm/filter-input.d/nginx_accept` and it will contain the rules necessary to open the `http` and `https` ports.

## Installation

`$ ansible-galaxy install nickjj.ferm`

## Requirements

Tested on ubuntu 12.04 LTS but it should work on other versions that are similar.

## Ansible galaxy

You can find it on the official [ansible galaxy](https://galaxy.ansible.com/list#/roles/1077) if you want to rate it.

## License

MIT