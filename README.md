# Rucio - Monitoring

This repository is to act as a beginners guide and collection of monitoring solutions for Rucio to build upon those found in the [Rucio Documentation](https://rucio.cern.ch/documentation/operator/monitoring)

To facilitate this a collection of helm-charts to deploy monitoring solutions for Rucio is included.
As well as the helm charts there is the READMEs that are to provide some documentation for each of the deployment methods to deploy and connect the components.
And finally json files to be imported into Grafana to produce some initial monitoring dashboards for you to work from and modify for your needs.

The intention of this repo is to allow people to configure and setup monitoring for their own Rucio instances, and when they have developed new dashboards and visualisations that they think could be useful to other communities contribute back.

Current contents include two types of monitoring in addition to the prometheus / Graphite monitoring:
-    Event monitoring from Hermes daemon
-    Database monitoring using Logstash and database queries.

To get these two types of monitoring this repository includes:

- Helm charts to deploy OpenSearch, the instructions to connect the components, and the dashboard to deploy on grafana to visualise the data from Rucio.
- Helm charts to deploy ElasticSearch, the instructions to connect the components, and the dashboard to deploy on grafana to visualise the data from Rucio.
- Helm charts to deploy Logstash, the instructions to connect the components and dashboards to deploy to OpenSearch and ElasticSearch
