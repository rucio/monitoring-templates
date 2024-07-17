# ElasticSearch monitoring

Within this directory are the resources to setup and use an ElasticSearch deployment, the instructions on how to connect Rucio to said ElasticSearch deployment, and how to register the ElasticSearch deployment as a datasource in grafana.

In brief, the deployment process goes as follows:
1. Use the helm chart to deploy an instance of ElasticSearch
2. Add the index to ElasticSearch to accept the messages from Hermes
3. Configure Hermes to have ElasticSearch as its elastic_endpoint
4. Register ElasticSearch as a datasource in Grafana
5. Deploy dashboards to Grafana to visualise the data

## 1. Use the helm chart to deploy an instance of ElasticSearch

There are a number of different ways to utilise the charts. Here I will demonstrate the process of using a make file to take in the values.yaml and produce a helm file with all of the components described and use helm to deploy the components. You can use the helm file to do the same or use a CI/CD pipeline, or whatever method you use.

For this system you will need to deploy the custom resource definitions and elastic operator that is found in the [elastic search documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html) 

The default settings for ElasticSearch are to have 3 master Nodes and 3 worker nodes.

Ensure you have the elasticsearch repo added to your helm repo list

> `helm repo add elastic https://helm.elastic.co`
> `helm repo update`

Modify the `values-elasticsearch.yaml` to meet your deployment needs, whether this is the certificate useage for the pod, ingress, services, node count and capabilities/roles, setting the vm.max_map_count in one of the various ways (this needs setting to 262144) for the elasticsearch containers to run.

Once you have modified the `values` and you are content to attempt to deploy,

navigate a terminal to the directory with the `Makefile` for ElasticSearch

In the terminal run:
> `make elastic-deployment`
to create the file `helm-elasticsearch.yaml` using the values you have in the directory and the templates that are part of the [Cloud-on-k8s](https://github.com/elastic/cloud-on-k8s/tree/main) repository.
The `Makefile` also edits the produced helm file to run on a basic license, this can of course be removed should you be using an enterprise license.

To deploy this helm file you can then run 
> `helm install elasticsearch elastic/elasticsearch -n <namespace> -f ./values-elasticsearch.yaml`


## 2. Add the index to ElasticSearch to accept the messages from Hermes

Adding the index to the ElasticSearch can be completed in two ways: using curl, and the ElasticSearch CLI. Here I will cover using curl, adding indexes using the ElasticSearch CLI can be found in their [documentation](https://ElasticSearch.org/docs/latest/api-reference/index-apis/create-index/).

### A. Utilising curl

#### Get access to ElasticSearch:

Ensure you have the correct credentials by running the following command in one terminal 

> `kubectl get secret elasticsearch-eck-elasticsearch-es-elastic-user kubectl get secret -n elastic-system elasticsearch-eck-elasticsearch-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'`

Then portforward to the service to get access to the deployment

> `kubectl port-forward -n elastic-system svc/elasticsearch-eck-elasticsearch-es-worker 9200`

In another terminal run, the default username and password is admin:

> `curl -u "<ElasticSearch username>:<ElasticSearch password>" -k "https://localhost:9200"`

Should all be correct you will get a response that shows:



>{  
>  "name" : "elasticsearch-eck-elasticsearch-es-worker-0",  
>  "cluster_name" : "elasticsearch-eck-elasticsearch",  
>  "cluster_uuid" : "wz8ZdQf1RjqHA68Ba8WzLA",  
>  "version" : {  
>    "number" : "8.11.0",  
>    "build_flavor" : "default",  
>    "build_type" : "docker",  
>    "build_hash" : "d9ec3fa628c7b0ba3d25692e277ba26814820b20",  
>    "build_date" : "2023-11-04T10:04:57.184859352Z",  
>    "build_snapshot" : false,  
>    "lucene_version" : "9.8.0",  
>    "minimum_wire_compatibility_version" : "7.17.0",  
>    "minimum_index_compatibility_version" : "7.0.0"  
>  },  
>  "tagline" : "You Know, for Search"  
>}  

#### Create index in ElasticSearch
Within the Repo from step 1 at `overlays/Elastic/rucio-events` contains the index, this needs to be applied to ElasticSearch in the location you want the index to be created 

> `curl -u "elastic:<Elasticsearch password>" -XPUT "https://localhost:9200/<you Index>/?&pretty" -H "Content-Type: application/json" -d @rucio-events -k`


## 3. Configure Hermes to have ElasticSearch as its elastic_endpoint

ElasticSearch is now setup to receive messages from Rucio. It now needs to have Rucio Hermes daemon pointed at it. For this editing of the Rucio values will allow for such thing. 

You will need at least this in the config section of the daemon values 

>  hermes:  
>    services_list: "elastic"  
>    elastic_endpoint: "http://elasticsearch-eck-elasticsearch-es-worker.elastic-system.svc.cluster.local:9200/<your index>/_bulk"  

You will also need to create a secret in the Rucio namespace to allow the injection of the ElasticSearch user secret for Hermes to be able to deposit the messages 


`kubectl create secret generic <daemons deploymentname>-search-secrets --namespace <Rucio namespace> --from-literal=USERNAME=<username> --from-literal=PASSWORD=<Elasticsearch password>`

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


## 4. Register ElasticSearch as a datasource in Grafana

Now that Rucio is now communicating its logs to ElasticSearch it now needs to be able to be visualised using Grafana

In your clusters Grafana deployment go to

- configuration / datasources

- Add data source

- Select ElasticSearch

- Put in the following details in the fields: 

>URL: http://ElasticSearch-eck-ElasticSearch-es-worker.elastic-system.svc.cluster.local:9200  
>Basic auth: True  
>User: <USERNAME>  
>Password: <ElasticSearch password>  
>Index name: <your Index>  
>Pattern: No pattern  
>Time field name: create_at  
>Max concurrent Shard Requests: 3 # If you have used the base repo settings  


## 5. Deploy dashboards to Grafana to visualise the data

Once the data source is created and verified you can create a dashboard using it

Navigate to Dashboards / Import

Paste in the contents of `overlays/Elastic/Dashboards/Rucio External` into the import via panel json