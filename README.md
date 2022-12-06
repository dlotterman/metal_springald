# metal_springald

`metal_springald` is intended to be used as a host / worker harness for benchmarking / load generation activities inside of Equinix Metal.

Currently under active development, `metal_springald` is intended to:

- Provision Equinix Metal instances with a base Linux OS (Ubuntu, CentOS etc), where those instances are intended to be used as load generation hosts.
- Configure those hosts appropriately, including both Equinix Metal platform and the host OS itself, so that worker instances can join and participate in the correct network namespaces
- Monitor those load generation (and other) worker hosts with [prometheus](https://prometheus.io/docs/introduction/first_steps/) + [node_exporter](https://prometheus.io/docs/guides/node-exporter/) + [grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)

![](https://s3.us-east-1.wasabisys.com/metalstaticassets/springald.PNG)

## Setup

### Getting Started

Before beginning, it is strongly recomemended that an operator complete the [Equinix Metal Getting Started](https://metal.equinix.com/developers/docs/) guide before proceeding with anything referenced here. The getting started documentation will make sure the account is correctly setup and walk the operator through the basics of operating Metal.

### Required Accounts and Credentials:

- [Equinix Metal Read / Write Token](https://github.com/openshift/assisted-installer)
- [Equinix Metal Project ID](https://metal.equinix.com/developers/docs/accounts/projects/)

#### Environment Setup

- Clone this repository
  - `git clone https://github.com/dlotterman/metal_springald`
- Create Python [virtualenv](https://docs.python.org/3/library/venv.html) (Python 3.9 or newer is expected)
  - `python3 -m venv metal_springald`
- Change directory into the repository
  - `cd metal_springald`
- Source the [virtualenv](https://docs.python.org/3/library/venv.html#how-venvs-work) environment setup / activate
  - `source bin/activate`
- Update pip
  - `pip install --upgrade pip`
- Install Python packages required
  - `pip install -r requirements.txt`
- Install Equinix Metal Ansible Galaxy Collection
  - `ansible-galaxy collection install equinix.metal`
- Export correct environment variables:
  - `export METAL_API_TOKEN=$YOUR_METAL_API_KEY_HERE`
  - `export METAL_PROJ_ID=YOUR_METAL_PROJECT_ID_HERE`
  
#### Configuration Editing

The only **required** edit is to add your Metal Project ID to the Ansible [Inventory Plugin file](https://github.com/dlotterman/metal_springald/blob/d8336544d88e47430b7f3a29fcd3a81574f7e713/equinix_metal.yaml#L15).

Otherwise the majority of the availaible configuration should be in the [all.yaml](https://github.com/dlotterman/metal_springald/blob/main/group_vars/all.yaml) file in the `group_vars` directory.

Note that the environment variable exports from the end of the Environment Setup and the sourcing of the `bin/activate` venv file **MUST** be performend with every bash / user session. I.E if you logout and log back into your shell, you must re-export and activate those environment settings.

#### Installing node_exporter and configuring prometheus

Applying the tag `metal_springald` to an instance, including one node provisioned by this harness, will trigger this harness to try to install `node_exporter` as a `systemd` service to that host.

Any host with the `metal_springald` tag will also be added to the Prometheus configuration that will be instantiated on the first client host.

#### Configuring Grafana

Grafana (+Prometheus) gets setup without any configuration on the first client (also called worker) on port `3000`. Once the Ansible `metal_springald_provision.yaml` is run, Grafana should be available at: `http://client01-IP:3000`, where the initial user + password will be `admin` + `admin`. 

Once logged in: 
- The Prometheus datasource can be added with the default host string of `http://localhost:9090`
- A pre-built `node_exporter` dashboard can be imported, an example one [here](https://grafana.com/grafana/dashboards/1860-node-exporter-full/), whose Grafana ID is `1860`

#### Confirmation

If everything is setup correctly, you should be able to run `ansible-inventory -i equinix_metal.yaml --list` and the command will return **empty** but *without errors*.

With errors:
```
$ ansible-inventory -i equinix_metal.yaml --list
[WARNING]:  * Failed to parse /home/adminuser/code/metal_assisted_openshift/equinix_metal.yaml with auto plugin: Failed
to query devices from Equinix Metal API. Error 404: Not found
[WARNING]:  * Failed to parse /home/adminuser/code/metal_assisted_openshift/equinix_metal.yaml with yaml plugin: Plugin
configuration YAML file, not YAML inventory
[WARNING]:  * Failed to parse /home/adminuser/code/metal_assisted_openshift/equinix_metal.yaml with ini plugin: Invalid
host pattern '---' supplied, '---' is normally a sign this is a YAML file.
[WARNING]: Unable to parse /home/adminuser/code/metal_assisted_openshift/equinix_metal.yaml as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    }
}
```

*Without* errors:
```
$ ansible-inventory -i equinix_metal.yaml --list
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "equinix_metal",
            "ungrouped"
        ]
    }
}
```
#### iperf3 as an example workload / orchestration

The orchestration of the provisioned workers is managed through Ansible. Ansible's `free` [strategy](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html) is what will trigger the workload generation across all hosts at the same time. The invocation of that can be seen [here](https://github.com/dlotterman/metal_springald/blob/d8336544d88e47430b7f3a29fcd3a81574f7e713/metal_springald_siege.yaml#L5), where the `free` strategy means that all tasks in the `play_siege.yaml` playbook will be executed in parralel.

`iperf3` is used as the example in this repository, where the configuration of the `iperf3` server is run on an instance configured outside of this repo, but the configuration of the test of that endpoint can be orchestrated as seen in the [all.yaml](https://github.com/dlotterman/metal_springald/blob/d8336544d88e47430b7f3a29fcd3a81574f7e713/group_vars/all.yaml#L26) file.

### Provisioning the environment

This is the fun part:

- `ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i equinix_metal.yaml -u root metal_springald_provision.yaml` will provision the environment as configured (and auto accept the Metal hosts SSH keys)
- `ansible-playbook -i equinix_metal.yaml -u root metal_springald_siege.yaml` will perform the `siege`