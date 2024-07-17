# OpenSearch Monitoring

The OpenSearch requirements and configuration parameters are described in their [github repository](https://github.com/opensearch-project/helm-charts/tree/main/charts/opensearch), but the values files included here with a small amount of edits will deploy a 3 master node, and 3 worker nodes, with persistent storage with the default username and password, and not externally accessible.

The configuration described within this directory provides a basic monitoring setup to be built upon. The Rucio messaging daemon delivers messages to the index, this index will grow over time and can become unmanageable. It would be best to investigate [OpenSearch Index State Management (ISM)](https://opensearch.org/docs/latest/im-plugin/ism/index/) to create a data retention policy for your logs, utilising the rollover action to split the logs into daily indexes, and apply a retention policy.

This Helm chart installs [OpenSearch](https://github.com/opensearch-project/OpenSearch) with configurable TLS, RBAC and much more configurations. This chart caters a number of different use cases and setups.


## 1. Use the helm chart to deploy an instance of OpenSearch
The way this repository is setup, it uses a values file that the operators edit, that is then combined with remote templates in the [OpenSearch helm-charts repository](https://github.com/opensearch-project/helm-charts) to create a local helm file that can then be used by helm.
Install the Opensearch helm charts:
> `helm repo add opensearch https://opensearch-project.github.io/helm-charts/`
> `helm repo update`


### Edit the values file to your needs
The values file included will get a cluster started but it may not have all the features, or configuration you need for your situation. You should consult the [OpenSearch Documentation](https://opensearch.org/docs/latest/) and decide what you do need but below are some suggestions. Any changes you want on both workers and masters will need to be included in both values files.

#### vm.max_map_count settings

vm.max_map_count is a setting that needs to be set to 262144 on the nodes that host your OpenSearch cluster. To set it you can do it manually by adding:
> `vm.max_map_count=262144`
to the `/etc/sysctl.conf` file, and then reload the configuration with;
> `sudo sysctl -p`

Or if you are running your containers with privialages you can enable the sysctl.

While manual works for a miniKube or small cluster, and sysctl allows you to do it if you run privillaged containers, you should be aware that it can be set using initialisation containers (InitContainers), wihch is a short lived container that is run privillaged but once its job is done it closes.
This can be done by enabling `sysctlInit`

#### Nodes for the cluser
An initial 3 master and 3 worker nodes are configured, but you may wish to change this for your needs.

### Get access to OpenSearch:

Portforward to the service to get access to the deployment

> `kubectl port-forward -n opensearch-system svc/opensearch-cluster-worker 9200`

In another terminal run, the default username and password is admin:

> `curl -u "<OpenSearch username>:<OpenSearch password>" -k "https://localhost:9200"`

Should all be correct you will get a response that shows:


>{
>  "name" : "opensearch-cluster-worker-2",
>  "cluster_name" : "opensearch-cluster",
>  "cluster_uuid" : "EmhtKET_SLGrMEi8cSYdPw",
>  "version" : {
>    "distribution" : "opensearch",
>    "number" : "2.9.0",
>    "build_type" : "tar",
>    "build_hash" : "1164221ee2b8ba3560f0ff492309867beea28433",
>    "build_date" : "2023-07-18T21:23:29.367080729Z",
>    "build_snapshot" : false,
>    "lucene_version" : "9.7.0",
>    "minimum_wire_compatibility_version" : "7.10.0",
>    "minimum_index_compatibility_version" : "7.0.0"
>  },
>  "tagline" : "The OpenSearch Project: https://opensearch.org/"
>}

## 2. Add the index to OpenSearch to accept the messages from Hermes

Within the Repo from step 1 at `overlays/OpenSearch/rucio-events` contains the index, this needs to be applied to ElasticSearch in the location you want the index to be created 

> `curl -u "<username>:<OpenSearch password>" -XPUT "https://localhost:9200/<you Index>/?&pretty" -H "Content-Type: application/json" -d @rucio-events -k`


## 3. Configure Hermes to have OpenSearch as its elastic_endpoint

OpenSearch is now setup to receive messages from Rucio. It now needs to have Rucio Hermes daemon pointed at it. For this editing of the Rucio values will allow for such thing. 

You will need at least this in the config section of the daemon values 

>  hermes:  
>    services_list: "elastic"  
>    elastic_endpoint: "https://opensearch-cluster-worker.opensearch-system.svc.cluster.local:9200/<your index>/_bulk"  

You will also need to create a secret in the Rucio namespace to allow the injection of the OpenSearch user secret for Hermes to be able to deposit the messages 


`kubectl create secret generic <daemons deploymentname>-search-secrets --namespace <Rucio namespace> --from-literal=USERNAME=<username> --from-literal=PASSWORD=<Opensearch password>`

Add the secret to Hermes, an example is below

>hermes:  
>  threads: 1  
>  podAnnotations: {}  
>  bulk: 1000  
>  resources:  
>    limits:  
>      memory: "600Mi"  
>      cpu: "210m"  
>    requests:  
>      memory: "300Mi"  
>      cpu: "140m"  
>  - name: RUCIO_CFG_HERMES_ELASTIC_USERNAME  
>    valueFrom:  
>      secretKeyRef:  
>        name: search-secrets  
>        key: USERNAME  
>  - name: RUCIO_CFG_HERMES_ELASTIC_PASSWORD  
>    valueFrom:  
>      secretKeyRef:   
>        name: search-secrets  
>        key: PASSWORD  

## 4. Register OpenSearch as a datasource in Grafana

Now that Rucio is now communicating its logs to OpenSearch it now needs to be able to be visualised using Grafana

In your clusters Grafana deployment go to

- configuration / datasources

- Install the OpenSearch datasource plugin

- Add data source

- Select OpenSearch

- Put in the following details in the fields: 

> URL: https://opensearch-cluster-worker.opensearch-system.svc.cluster.local:9200  
> Basic auth: True  
> User: <USERNAME>  
> Password: <OpenSearch password>  
> Index name: <your Index>  
> Pattern: No pattern  
> Time field name: create_at  
> Max concurrent Shard Requests: 3 # If you have used the base repo settings  

## 5. Deploy dashboards to Grafana to visualise the data

Once the data source is created and verified you can create a dashboard using it

Navigate to Dashboards / Import

Paste in the contents of `overlays/OpenSearch/Dashboards/Rucio External` into the import via panel json
