<!-- .slide: data-state="section-break" id="section-break-1" data-timing="10s" -->
# Salting Your VMs

Note:
* Software Defined VMs


<!-- .slide: data-state="normal" id="salt-states" data-timing="240s" data-menu-title="Salt States" -->
## Salt States

* Define the target setup:

```yaml
install_webserver:
  pkg:
    - installed
      - name: apache2
```

* Apply on multiple minions
    * `salt 'web-*' state.apply webserver`
    * `/etc/salt/top.sls`

```yaml
base:
  'web-*':
    - webserver
```

https://docs.saltstack.com/en/latest/ref/states/all/index.html

Note:
* Lots of shipped states
* ``/srv/salt/top.sls``
    * patterns to select minions and apply state files
    * GitOps


<!-- .slide: data-state="normal" id="vm-definition-state" data-timing="120s" data-menu-title="VM Definition" -->
## VM definition

```yaml
srv01:
  virt.running:
    - name: srv01
    - cpu: 1
    - mem: 1024
    - disks:
      - name: system
        model: virtio
        format: qcow2
        image: https://url.to/a/template.qcow2
        pool: pool0
        size: 122880
    - interfaces:
      - name: eth0
        type: network
        source: net0
    - graphics:
        type: vnc
    - seed: False
```

Note:
* image === template
* size: uses ``virt-resize``
* disks: need to use pools
* Disk size and mem: in MiB
* seed: injecting Salt


<!-- .slide: data-state="normal" id="pool-definition" data-timing="60s" data-menu-title="Pool Definition" -->
## Storage pool definition
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
```

Fixes on the way!

Note:
* support for all libvirt types
* libvirt XML data as YAML!


<!-- .slide: data-state="normal" id="net-definition" data-timing="120s" data-menu-title="Net Definition" -->
## Network definition
```yaml
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
```

Note:
* Bridged:✔
* NAT: patch on the way
* Others: ❌
* Missing IP/DNS fine tuning


<!-- .slide: data-state="normal" id="chaining" data-timing="60s" data-menu-title="Putting it together" -->
## Putting it together

```yaml
srv01:
  virt.running:
    ...
    - require:
      - virt: pool0
      - virt: net0
```

https://docs.saltstack.com/en/latest/ref/states/requisites.html


<!-- .slide: data-state="normal" id="templating" data-timing="120s" data-menu-title="To The Next Level" -->
## To The Next Level

```yaml
{{pillar['vm_name']}}:
  virt.running:
    ...
    - require:
      - virt: pool0
    {% if "vm_net" in pillar %}
      - virt: {{pillar['vm_net']}}
    {% endif %}
```

* https://docs.saltstack.com/en/latest/ref/renderers/
* https://docs.saltstack.com/en/latest/topics/pillar/

<div style="width:30%; margin-left: auto; margin-right: auto; margin-top: 2em">
<img alt="git logo" data-src="images/git_icon.svg" style="display: block; float: left"/>
<img alt="git logo" data-src="images/cog.svg" style="display: block; float: right"/>
</div>

Note:
* Jinja, renderers
* Pillar data
