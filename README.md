# vDU profile testing

vDU makes use of a SNO OpenShift deployment.

In this case, we want to define an hyperconverged cluster with 3 master nodes and no workers

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
watch -d 'oc get clusterversion; oc get nodes; oc get co'
```

## Configure the cluster

According to the [documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/scalability_and_performance/index#telco-ran-du-reference-design-components_telco-ran-du) we've several components required for our system:
