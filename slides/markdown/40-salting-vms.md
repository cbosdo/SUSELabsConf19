<!-- .slide: data-state="section-break" id="section-break-1" data-timing="10s" -->
# Salting Your VMs


<!-- .slide: data-state="normal" id="salt-states" data-timing="20s" data-menu-title="Salt States" -->
## Salt States

* Define the target setup:

```yaml
install_webserver:
  pkg:
    - installed
      - name: apache2
```

* Apply on multiple minions
    * `salt 'web-*' state.apply install_webserver`
    * `top.sls` file:

```yaml
base:
  'web-*':
    - webserver
```


<!-- .slide: data-state="normal" id="vm-definition-state" data-timing="20s" data-menu-title="VM Definition" -->
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


<!-- .slide: data-state="normal" id="pool-definition" data-timing="20s" data-menu-title="Pool Definition" -->
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


<!-- .slide: data-state="normal" id="net-definition" data-timing="20s" data-menu-title="Net Definition" -->
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


<!-- .slide: data-state="normal" id="chaining" data-timing="20s" data-menu-title="Putting it together" -->
## Putting it together

* Chaining it:

```yaml
srv01:
  virt.running:
    ...
    - require:
      - virt: pool0
      - virt: net0
```

* Template it
* Store in Git
* Automate!
