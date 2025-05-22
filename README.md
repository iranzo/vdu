# vDU profile testing

vDU makes use of a SNO OpenShift deployment.

In this case, we want to define an hyperconverged cluster with 3 master nodes and no workers.

## Requirements

- KCLI client
- Working pull secret

## Setting up the infrastructure

Create a `kcli_plan.yaml`:

```yaml
parameters:
  cluster: cluster
  tag: '4.18'
  version: 'stable'
  domain: karmalabs.corp
  number: 0
  network: default
  cidr: 192.168.122.0/24

{{ network }}:
  type: network
  cidr: {{ cidr }}
  disks:
   - size: 100
   - size: 100


{% set num = 0 %}

{% set api_ip = cidr|network_ip(200 + num ) %}

{% set cluster = 'hub' %}

hub:
  type: cluster
  kubetype: openshift
  domain: {{ domain }}
  ctlplanes: 3
  api_ip: {{ api_ip }}
  numcpus: 16
  memory: 32768

api-hub:
 type: dns
 net: {{ network }}
 ip: {{ api_ip }}
 alias:
 - api.{{ cluster }}.{{ domain }}
 - api-int.{{ cluster }}.{{ domain }}

{% if num == 0 %}
apps-hub:
 type: dns
 net: {{ network }}
 ip: {{ api_ip }}
 alias:
 - console-openshift-console.apps.{{ cluster }}.{{ domain }}
 - oauth-openshift.apps.{{ cluster }}.{{ domain }}
 - prometheus-k8s-openshift-monitoring.apps.{{ cluster }}.{{ domain }}
 - canary-openshift-ingress-canary.apps.{{ cluster }}.{{ domain }}
 - multicloud-console.apps.{{ cluster }}.{{ domain }}
{% endif %}


{% for num in range(1, number) %}
{% set api_ip = cidr|network_ip(200 + num ) %}
{% set cluster = "cluster" %}

cluster{{ num }}:
  type: cluster
  kubetype: openshift
  domain: {{ domain }}
  ctlplanes: 1
  api_ip: {{ api_ip }}
  numcpus: 16
  memory: 18384

api-cluster{{ num}}:
 type: dns
 net: {{ network }}
 ip: {{ api_ip }}
 alias:
 - api.{{ cluster }}{{ num }}.{{ domain }}
 - api-int.{{ cluster }}{{ num }}.{{ domain }}

{% endfor %}

```

Store your pull secret in a file named `openshift_pull.json` alongside above plan.

## Create the cluster

Execute `kcli create plan`.

For monitoring the creation you can use a commandline like (as well as the `kcli` command output being printed on screen):

```sh
KUBECONFIG=/root/.kcli/clusters/hub/auth/kubeconfig
watch -d 'oc get clusterversion; oc get nodes -o wide; oc get mcp;oc get co'
```

## Configure the cluster

According to the [documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/scalability_and_performance/index#telco-ran-du-reference-design-components_telco-ran-du) we've several components required for our system:

![DU Profile schema](du.png)

Inside the `manifests` folder there's a `kustomization.yaml` so that we can use `oc apply -k .` to get all manifests applied to our cluster.

Note that some of the changes, are configuring the requirement for `kernel-rt` so the `MachineConfigurationPools` will be updated and once the update is finished, it will be applied to the nodes that will reboot to apply the configuration changes.

At the end of the process, `oc get nodes -o wide` will list as kernel version the one ending in `+rt`like:

```sh
$ oc get nodes -o wide
NAME                            STATUS   ROLES                         AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                                                KERNEL-VERSION                    CONTAINER-RUNTIME
hub-ctlplane-0.karmalabs.corp   Ready    control-plane,master,worker   5d22h   v1.31.8   192.168.122.8     <none>        Red Hat Enterprise Linux CoreOS 418.94.202505062142-0   5.14.0-427.68.1.el9_4.x86_64+rt   cri-o://1.31.8-3.rhaos4.18.gitf0f6e96.el9
hub-ctlplane-1.karmalabs.corp   Ready    control-plane,master,worker   5d22h   v1.31.8   192.168.122.228   <none>        Red Hat Enterprise Linux CoreOS 418.94.202505062142-0   5.14.0-427.68.1.el9_4.x86_64+rt   cri-o://1.31.8-3.rhaos4.18.gitf0f6e96.el9
hub-ctlplane-2.karmalabs.corp   Ready    control-plane,master,worker   5d22h   v1.31.8   192.168.122.97    <none>        Red Hat Enterprise Linux CoreOS 418.94.202505062142-0   5.14.0-427.68.1.el9_4.x86_64+rt   cri-o://1.31.8-3.rhaos4.18.gitf0f6e96.el9
```

If we want to update our cluster (via `oc adm upgrade`) we might need to acknowledge the changes to API via:

```sh
$ oc -n openshift-config patch cm admin-acks --patch '{"data":{"ack-4.18-kube-1.32-api-removals-in-4.19":"true"}}' --type=merge
```
