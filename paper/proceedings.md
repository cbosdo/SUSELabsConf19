# Introduction #


The original use case that needed to be addressed is to be able to manage Xen or KVM virtual machines using a web interface.
Of course there are already a number of products in that area like oVirt, ProxMox, Xen Orchestra or Kimchi but none was suiting the these criteria:

* simple enough to be usable at a small scale
* able to handle both KVM and Xen hypervisors
* have a live community

On the other hand there was a web product in the SUSE portfolio that already had some virtualization management capabilities: SUSE Manager.
As of now, SUSE Manager can be used for small scale setups, even though this story could be improved.
The virtualization features it already supports are using libvirt in a hypervisor-agnostic way.
There is no SUSE Manager / Uyuni live community outside of SUSE yet, but at least the foundation for the products where already here and maintained: no need to duplicate this effort.

# Implementation big picture #

Shedding some light on how SUSE Manager is implemented is needed in order to fully understand what has been done and what is still left.

## Tradition vs Salt ##

SUSE Manager used to be a downstream of an old and now less active project: spacewalk (https://spacewalkproject.github.io).
However since May 2018 it has been open sourced into a new project: Uyuni (https://www.uyuni-project.org).
This has some consequences on the whole project architecture, with a lot of legacy code being around.

SUSE Manager 3 introduced Salt.
This means that from that point on there are two types of managed systems: the ones managed using salt or the ones managed in the old way.
Those are commonly refered to as *Salt* or *Traditional* systems.

All of the following will focus on the Salt systems only, but it is important to remember that all virtualization features available in SUSE Manager were initially only for the traditional systems.

From a ten thousand feet perspective the principles of the traditional architecture to manage virtual machines is the same than the Salt one.
Some code runs on the hypervisor to manage the VMs and report their status.
The main benefit of using salt is to allow easy use of the low level bricks to customers.

## Assembling the pieces ##

SUSE Manager is web application written in Java, handling both the core of the logic and the display.

In order to get quick updates of the UI a salt engine reports the libvirt events to Salt.
SUSE Manager being a Salt Reactor, the event is consumed directly and the database is then updated with the latest VM changes.
In the following diagram, the engine part is a Salt engine named `libvirt_events` that registers a libvirt event callback converting all libvirt events into Salt events.
In some cases, like a VM creation, the libvirt event doesn't contain enough information for SUSE Manager to update its database and then a call to the salt virt module is made to get the missing VM definition.

![libvirt event handling](images/libvirt-event-chain.svg)

This allows the user to still manage the virtual machines using `virsh`, `virt-manager` or anything else if needed.
It also improves a lot the usability over the 5 minutes delay schedule for the `virt-poller` cron job used for traditional systems.
However the `virt-poller` job is still used in case something breaks to ensure the data are updated at least every five minutes.

Most of the actions from either the Web UI or the command line tools will be converted into an entry in the postgresql database.
This entry is later on picked up by a task scheduler called Taskomatic that requests the Salt master to apply the appropriate Salt state.
The Salt master then delegates this to the minion.
In the virtualization case, the Salt minion uses libvirt python API to do the job. The following diagram gives the example of a VM start:

![VM start action details](images/suma-virt-action.svg)

The idea behind this design is to keep all the virtualization handling code in Salt's code base.
This will provide powerfull features to be detailed in the next section.

## Leveraging Salt power for VMs ##

Salt allows describing a target configuration using YAML files.
These files are called state files and when applying them on a system Salt ensures the system will be in the described state and do the necessary changes.
We can imagine describing which virtual machines are needed, their setup and what they depend on in a state.

**Note:** in order for the example to run, the openSUSE salt 2019.2.0 package plus a few other patches from salt pull requests [#54196](https://github.com/saltstack/salt/pull/54196) and [#54196](http://github.com/saltstack/salt/pull/54197) is required.

Create a `/srv/salt/vm.sls` file with the following content:

```yaml
pool0:
  virt.pool_running:
    - name: pool0
    - ptype: netfs
    - target: /vms
    - source:
        dir: /srv/vms
        hosts:
          - mirror.tf.local
        format: nfs
    - autostart: True

net0:
  virt.network_running:
    - name: net0
    - bridge: net0-br
    - forward: nat
    - ipv4_config:
        cidr: 192.168.44.0/24
        dhcp_ranges:
          - start: 192.168.44.10
            end: 192.168.44.25
    - autostart: True

srv01:
  virt.running:
    - name: srv01
    - cpu: 1
    - mem: 1024
    - disks:
      - name: system
        model: virtio
        format: qcow2
        image: https://download.opensuse.org/repositories/systemsmanagement:/sumaform:/images:/libvirt/images/opensuse151.x86_64.qcow2
        pool: pool0
        size: 122880
    - interfaces:
      - name: eth0
        type: network
        source: net0
        mac: 2A:C3:A7:A6:01:04
    - graphics:
        type: vnc
    - seed: False
    - require:
      - virt: pool0
      - virt: net0
```

This state can either be applied using the `/srv/salt/top.sls` file as described in the [documentation](https://docs.saltstack.com/en/latest/topics/tutorials/states_pt1.html) or using the salt command as follows:

```
salt '*min-kvm*' state.apply vm
```

In the example a `srv01` virtual machine is started as well as a `net0` NAT network and a `pool0` NFS storage pool.
The state does not need to care whether those are already existing or not: it will define them and start them as needed.

The domain start up is also having dependencies on the storage pool and network as indicated by:

```yaml
    - require:
      - virt: pool0
      - virt: net0
```

All the other parts of the example state file are just describing what needs to be started.
Ideally Salt should update an existing pool, network or virtual machine that doesn't fit the description, but this is currently only implemented for the VMs.
For more details on the structure of virtualization-related states, check the [Salt virt states documentation](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.virt.html).

# Conclusion #

The example mentioned here is rather small and simple, but more complex ones could be crafted using multiple base images, interconnected virtual machines, etc.
Combined with the power of [Salt formulas](https://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html) it could also help defining virtual machines templates to be easily configured by end users through SUSE Manager [Formulas with forms](https://www.suse.com/documentation/suse-manager-3/3.2/susemanager-best-practices/html/book.suma.best.practices/best.practice.salt.formulas.and.forms.html) feature.
And of course it is another step towards the GitOps trend.
