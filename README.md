Bookstore Demo for OSM
======================

Based upon the demo Open Service Mesh / Getting Started / Deploy Applications](https://release-v1-2.docs.openservicemesh.io/docs/getting_started/install_apps/).

Prerequisites
-------------

* [Create](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli) an AKS cluster
* [Install]((https://learn.microsoft.com/en-us/azure/aks/open-service-mesh-about#installation-and-version)) the OSM add-on
* [Install](https://learn.microsoft.com/en-us/azure/aks/open-service-mesh-binary?pivots=client-operating-system-linux#download-and-install-the-open-service-mesh-osm-client-binary) the OSM CLI and configure it for [AKS mode](https://learn.microsoft.com/en-us/azure/aks/open-service-mesh-binary?pivots=client-operating-system-linux#configure-osm-cli-variables-with-an-osm_config-file)

Install and configure OSM CLI
-----------------------------

```sh
cat << EOF > $HOME/.osm/config.yaml
install:
  kind: managed
  distribution: AKS
  namespace: kube-system
EOF

# Specify the OSM version that matches your OSM add-on version
OSM_VERSION=$(kubectl get deploy osm-controller -n kube-system -o=jsonpath='{.spec.template.metadata.labels.app\.kubernetes\.io\/version}')

curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -
sudo mv ./linux-amd64/osm /usr/local/bin/osm
sudo chmod +x /usr/local/bin/osm
rm -rf ./linux-amd64
osm version
```

Optional: Install Kubeview
--------------------------

```sh
helm upgrade \
    --install kubeview \
    https://github.com/benc-uk/kubeview/releases/download/0.1.31/kubeview-0.1.31.tgz \
    -n kubeview \
    --create-namespace \
    --set loadBalancer.enabled=false

nohup kubectl port-forward -n kubeview svc/kubeview 8088:80 &
```

View KubeView at: http://localhost:8088

Install the Bookstore app
--------------------------

```sh
# Verify your mesh has permissive mode enabled
kubectl get meshconfig osm-mesh-config -n kube-system -o=jsonpath='{$.spec.traffic.enablePermissiveTrafficPolicyMode}'
# If not, then enable it
kubectl patch meshconfig osm-mesh-config -n kube-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}' --type=merge

kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse

# When you add namespaces to the OSM mesh, the OSM controller automatically injects
# the Envoy sidecar proxy containers with applications deployed in those namespaces.
osm namespace add bookstore bookbuyer bookthief bookwarehouse

kubectl describe ns bookstore

# Deploy the sample application to the AKS cluster
SAMPLE_VERSION=release-$(echo $OSM_VERSION | sed 's/\(v[0-9]\.[0-9]\)\.[0-9]/\1/')
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/apps/bookbuyer.yaml
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/apps/bookthief.yaml
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/apps/bookstore.yaml
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/apps/bookwarehouse.yaml
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/apps/mysql.yaml

kubectl get pods,deployments,serviceaccounts -n bookbuyer
kubectl get pods,deployments,serviceaccounts -n bookthief

kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookstore
kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookwarehouse

nohup kubectl port-forward $(kubectl get pod -n bookbuyer -o name) -n bookbuyer 8080:14001 &
#http://localhost:8080

nohup kubectl port-forward $(kubectl get pod -n bookthief -o name) -n bookthief 8083:14001 &
#http://localhost:8083

nohup kubectl port-forward $(kubectl get pod -n bookstore -o name) -n bookstore 8084:14001 &
#http://localhost:8084
```

Turn off permissive traffic and explicit allow bookbuyer to access bookstore
----------------------------------------------------------------------------

```sh
kubectl patch meshconfig osm-mesh-config -n kube-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge

# Apply an SMI traffic access policy for buying books
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/access/traffic-access-v1.yaml
```

Examine the access policies in the YAML file above.

The bookbuyer should now be purchasing books again.

Traffic Split with SMI
----------------------

```sh
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/apps/bookstore-v2.yaml

kubectl get pod,svc -n bookstore

nohup kubectl port-forward $(kubectl get pod -n bookstore -o name --selector app=bookstore,version=v2) -n bookstore 8082:14001 &
#http://localhost:8082

# Traffic is both to both bookstore v1 and b2 since the service/bookstore does not include a version label in the selector

kubectl describe -n bookstore service/bookstore
kubectl describe -n bookstore service/bookstore-v1
kubectl describe -n bookstore service/bookstore-v2

# Split traffic, initially 100% to bookstore-v1
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/split/traffic-split-v1.yaml

kubectl describe trafficsplit bookstore-split -n bookstore
```

Also, check the KubeView for `bookstore` namespace.

Split 50-50 between v1 and v2:

```sh
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/split/traffic-split-50-50.yaml

kubectl describe trafficsplit bookstore-split -n bookstore
```

Split 100% to v2:

```sh
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/$SAMPLE_VERSION/manifests/split/traffic-split-v2.yaml

kubectl describe trafficsplit bookstore-split -n bookstore
```

To decommision bookstore-v1 you would delete the v1 deployment and service.

Enable metrics in OSM monitored namespaces for Azure Monitor
------------------------------------------------------------

```sh
osm metrics enable --namespace bookbuyer
osm metrics enable --namespace bookstore
osm metrics enable --namespace bookthief
osm metrics enable --namespace bookwarehouse

# see: https://learn.microsoft.com/en-us/azure/aks/open-service-mesh-integrations#metrics-observability
kubectl apply -f azmon-osm.yaml
```

View OSM metrics for the AKS cluster and run this query (may take 10-15 mins for logs to start appearing initially):

```kql
InsightsMetrics
| where Name contains "envoy"
| extend t=parse_json(Tags)
| where t.app == "bookbuyer"
| order by TimeGenerated
```

and

```kql
InsightsMetrics
| where Name contains "envoy"
| distinct Name
```

View the OSM monitoring report in the AKS cluster via the preview link: https://aka.ms/azmon/osmux

Go to your AKS cluster / Insights / Reports / OSM monitoring

It may take a few mins to see the metrics appear.

Configure observability metrics for your mesh
---------------------------------------------

Refer to: https://release-v1-2.docs.openservicemesh.io/docs/guides/observability/

Open Service Mesh (OSM) generates detailed metrics related to all traffic within the mesh. These metrics provide insights into the behavior of applications in the mesh helping users to troubleshoot, maintain, and analyze their applications.

Deploy and configure Prometheus server:

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install stable prometheus-community/prometheus
```

Prometheus metrics collection is enabled automatically on namespaces that have been onboarded via `osm namespace add ...`.

Make a copy of the Prometheus server YAML:

```sh
kubectl get configmap | grep prometheus
kubectl get configmap stable-prometheus-server -o yaml > cm-stable-prometheus-server.yml
cp cm-stable-prometheus-server.yml cm-stable-prometheus-server.yml.copy
```

Ammend the Prometheus server YAML (`cm-stable-prometheus-server.yaml`), by copying the `kubernetes-pods` job from this page: https://release-v1-2.docs.openservicemesh.io/docs/guides/observability/metrics/#bring-your-own

It should look like this:

```yaml
 ...snip...
 - honor_labels: true
      job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
  ...snip...
```

Apply the config change:

```sh
kubectl apply -f cm-stable-prometheus-server.yml
```

Verify Prometheus is correctly configured to scrape OSM mesh and API endpoints:

```sh
PROM_POD_NAME=$(kubectl get pods -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace default port-forward $PROM_POD_NAME 9090
```

Open a browser up to http://localhost:9090/targets

Search for "osm" on the page - you should find some metrics endpoints.

Enter the following Prom query at http://localhost:9090/graph to see successful http requests:

```promql
envoy_cluster_upstream_rq_xx{envoy_response_code_class="2"}
```

Deploy and configure Grafana:

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install osm-grafana grafana/grafana
```

Retrieve the default Grafana password:

```sh
kubectl get secret --namespace default osm-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Port forward to Grafana and log in (user: admin, password: see above):

```sh
GRAF_POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $GRAF_POD_NAME 3000
```

Add a datasource in Grafan for Prometheus:

* Configuration / Data Sources / Add data source ([link](http://localhost:3000/datasources/new))
* Select Prometheus
* Update URL: `http://stable-prometheus-server.default.svc.cluster.local`
* Click **Save & test**

Import OSM dashboard:

* Download JSON dashboards for Grafana from here: https://github.com/openservicemesh/osm/tree/release-v1.2/charts/osm/grafana/dashboards

```sh
git clone https://github.com/openservicemesh/osm
cd osm
git checkout $SAMPLE_VERSION
cd charts/osm/grafana/dashboards/
cp *.json <dest_dir>/grafana-dashboards/
```

* Click *`+`* / Import ([link](http://localhost:3000/dashboard/import))
* Select "Upload JSON file"
* Click **Load**
* Select your `Prometheus` data source (if prompted)
* Enter `kube-system` for the OSM namespace (if prompted)
* Click **Import**

You will now see the Grafana dashboards for OSM.

Configure distributed tracing
-----------------------------

```sh
kubectl create namespace jaeger

kubectl apply -f jaeger.yaml
kubectl apply -f jaeger-rbac.yaml
kubectl patch meshconfig osm-mesh-config -n kube-system -p '{"spec":{"observability":{"tracing":{"enable":true, "address": "jaeger.jaeger.svc.cluster.local"}}}}' --type=merge

JAEGER_POD=$(kubectl get pods -n jaeger --no-headers  --selector app=jaeger | awk 'NR==1{print $1}')
kubectl port-forward -n jaeger $JAEGER_POD  16686:16686
```

Browse to: http://localhost:16686/

Cleanup
-------

```sh
kubectl patch meshconfig osm-mesh-config -n kube-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}' --type=merge

pkill kubectl

# or to fully clean up thje app
kubectl delete ns bookstore
kubectl delete ns bookbuyer
kubectl delete ns bookthief
kubectl delete ns bookwarehouse

# uninstall 3pp components
helm del kubeview -n Kubeview
helm del osm-grafana
helm del stable
```

References
----------

* https://openservicemesh.io/
* https://release-v1-2.docs.openservicemesh.io/
