# Hermes Daemon based monitoring

This directory offers two options to deploy which collect and parse messages from the hermes daemon. 
Using either of these two methods allows for monitoring of events in Rucio as mentioned in the [Rucio documentation](https://rucio.cern.ch/documentation/operator/monitoring#transfer-monitoring).
This solution builds additional monitoring that are not part of the [internal monitoring](https://rucio.cern.ch/documentation/operator/monitoring#internal-monitoring) solution that uses Graphite or Prometheus.

The type of monitoring this solution provides is on Rucio transfer and deletion events. Using these metrics you can get a clear idea on storage end point usage and throughput, user usage, protocols that are used, and scopes.
Both of these solutions use lucene queries and variables to allow manipulation of the date to group the events in various ways to help identify issues.

Once messages are delivered to ElasticSearch or OpenSearch they are archived to the messages_history table, so if there is an issue messages can be recovered. The Rucio documentation describes using ActiveMQ as a component between Rucio and ElasticSearch, but with development of the code the ActiveMQ component is no longer needed. The deployment described here is as simple as I could make it for ease of deployment, not reliability or redundancy.

