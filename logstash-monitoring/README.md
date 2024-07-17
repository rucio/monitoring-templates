# Rucio Database based monitoring

Database based monitoring uses logstash to schedule, make queries from the Rucio database and send the results to Elastic/OpenSearch to be parsed for Visualisation.

Initial Rucio pipelines were developed by Li Teng and modified by Timothy Noble to be used in Rucio version 34 and created an easy to deploy version using helm and kustomise.

Within this directory are the resources to setup and use a logstash to deploy some database querying pipelines as the inputs and send the information to a XXXXSearch deployment as the output.

The pipelines as laid out within this directory are setup to create a new index daily, and to be amalgamated under an alias for data visualisation purposes. In addition for an alias, it is recommended that a data retention policy is created in the Search software you have chosen to use. For OpenSearch this is the [Index State Management](https://opensearch.org/docs/latest/im-plugin/ism/index/), and for ElasticSearch this is the [Index Lifecyle Management](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/index-lifecycle-management.html).

In brief. the deployment process goes as follows:
1. Setup helm repos
2. Edit the helm chart dependant on which Search software you have deployed
3. Deploy secrets to the cluster to be used by logstash to query the database and write to Search
4. Deploy Logstash using the helm chart
5. Register the various indexes as datasources in Grafana
6. Deploy dashboards to Grafana to visualise the data



## 1. Deploy an instance of Logstash

### a. Deployment of Logstash using helm / Makefile

Logstash is found in the elastic collection of tools:
> `helm repo add elastic https://helm.elastic.co`
> `helm repo update`

Modify the `values-logstash.yaml` to meet your deployment needs, whether this is the certificate useage for the pod, ingress, services, node count and capabilities/roles.

Once you are happy with the `values` navigate to the logstash directory to use the `Makefile` for logstash

in the terminal run:
> `make logstash`
to create the file `helm-logstash.yaml` using the values you have in the directory and the templates that are part of the [Cloud-on-k8s](https://github.com/elastic/cloud-on-k8s/tree/main) repository.

To deploy the helm file you just had created run:
> `helm install logstash elastic/logstash -n <namespace> -f ./values-logstash.yaml`

The deployment, if using the default setting from this repository will not deploy right away as there will be secrets missing which will be deployed in `section 2`.

### b. Configuration of the pipelines

Before creating the secrets there is the matter of configuring the logstash pipelines to your needs.
The default pipelines included utilise the secrets created in `section 2`. The parts that you may wish to change is the pipeline names, the ssl use, or certificate verification. 
By default the pipelines are named `rucio-<pipeline.id>-<DATE>.
This setup will create indexes each day and populate them throughout that day (if run more than once).
While this setup may appear messy at first, it does allow the use of  State/Lifecycle Management to keep data for a certain length of time before deleting it.
And with the use of aliases and templates within OpenSearch the dated indexes of each pipeline can be recalled with a single grafana connection for data visualisation.


## 2. Deploy secrets to the cluster to be used by logstash to query the database and write to Search

Logstash as it is setup with the values provided in this repository requires 3 secrets with 3 key-value pairs for 2 of them and a file for the other, to be included in the same namespace as the logstash deployment. These secrets are:
> `rucio-pipelines` # The secret that contains the pipelines.yml file in the `Pipelines` directory
> `es-secrets` # which includes 3 key value pairs, USER, PASSWORD, and HOSTS with HOSTS being the connection URL to either ElasticSearch or OpenSearch (depending on what you are using)
> `db-secrets` # which includes 3 key value pairs, USERNAME, PASSWORD, and CONNECT with the last being the connection url to your Rucio database


## 3. Register the various indexes as datasources in Grafana

Within the pipelines.yml you will configure pipelines that each will want to be registered into Grafana as connections.

In your clusters Grafana deployment go to

- configuration / datasources

- Add data source

- Select XXXXSearch

- Put in the following details for your Search product of choice, referencing the indexes you have configured in the `pipelines.yml` file in the index line in the output section for each pipeline.

> URL: https:/<XXXXSearchSoftware>.<XXXXSearchNameSpace>.local:9200  
> Basic auth: True  
> User: <USERNAME>  
> Password: <XXXXSearch password>  
> Index name: <your Index>  
> Pattern: No pattern  
> Time field name: create_at  
> Max concurrent Shard Requests: 3 # If you have used the base repo settings  


## 4. Deploy dashboards to Grafana to visualise the data

Once the data source is created and verified you can create a dashboard using it

Navigate to Dashboards / Import

Paste in the contents of `overlays/XXXXSearch/Dashboards/Rucio Storage` into the import via panel json

