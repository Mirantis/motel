# Mirantis OpenTelemery
This repo contains 3 charts to deploy a monitoring stack using HMC and get metrics into regional clusters, data from which is then aggregated into single grafana interface.
![alt text](motel-arch.png)

## Mothership cluster chart
* central grafana interface
* promxy to forward calls to multiple downstream regional metrics servers
* local victoriametrics storage for alerting record rules
* hmc helmchart definitions and service templates to deploy regional and child charts into managedclusters

### Demo deployment
In `demo/demo-mothership-values.yaml` set your target ingress names that you are going to use for your regional clusters, but they can always be changed after the fact
```
helm repo add motel https://mirantis.github.io/motel/
helm repo update
helm upgrade -i motel motel/motel-mothership -n hmc-system -f demo/demo-mothership-values.yaml
```

## Regional cluster chart
* Grafana - region-specific Grafana instance, deployed and configured with grafana-operator
* vmcluster - metrics storage, ingestion, querying
* vmlogs - logs storage
* vmauth - auth frontend for metrics and logs ingestion and query services

#### Cluster requirements
- cert-manager
- ingress-nginx

To deploy regional `managedcluster` configure desired ingress names for vmauth and regional Grafana in it's values for the `motel-regional` template.
`demo/cluster/aws-regional.yaml` contains example definitions
```
kubectl apply -f demo/cluster/aws-regional.yaml
# you can check helm chart deployment status using ClusterSummary object:
kubectl get clustersummaries.config.projectsveltos.io -n hmc-system
```
Once the regional managedcluster is ready - retrieve its kubeconfig and get loadbalancer IP/DNS name for your ingress-nginx service.
```
kubectl get secret -n hmc-system aws-reg0-kubeconfig -o jsonpath={.data.value} | base64 -d  > /tmp/hmc-aws-reg0-kubeconfig.yaml
export KUBECONFIG=/tmp/hmc-aws-reg0-kubeconfig.yaml
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

With your preffered DNS hosting, set your ingress domains to resolve to that IP/DNS name, that's how the traffic will flow to/from regional cluster. 
To simplify this process it is posssible to enable [external-dns](https://kubernetes-sigs.github.io/external-dns/) helm chart deployment in values.

Once your domain is resolvable your Grafana and vmauth should be accessible.

## Child cluster chart
* vmagent - scrapes prometheus targets and forwards metrics to regional VictoriaMetrics cluster
* fluentd - collects logs and forwards them to regional VictoriaLogs storage

`demo/cluster/aws-child.yaml` contains example definitions

To deploy child `managedcluster` configure ingress names for regional vmauth in its values for the `motel-child` template.

```
kubectl apply -f demo/cluster/aws-child.yaml
# you can check helm chart deployment status using ClusterSummary object:
kubectl get clustersummaries.config.projectsveltos.io -n hmc-system
```

Once your child cluster is up, it should start pushing metrics and logs to your regional one, through ingress domain you've configured.
Check your regional Grafana for results first, then you should be able to see the same cluster in Grafana on the "mothership".

### Scaling up
* Deploy more child clusters in a single region and point them to the existing regional victoria stack.
* Repeat the previous two steps for each desired region
* Update mothership chart configuration with every deployed regional stack to aggregate the data
