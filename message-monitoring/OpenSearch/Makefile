OPEN_VERSION := 2.13.0

helm:
	helm repo add elastic https://helm.elastic.co
	helm repo update opensearch


open-deployment:
	helm template opensearch opensearch/opensearch --version=${OPEN_VERSION} --values=values-opensearch-master.yaml > helm-opensearch-master.yaml
	helm template opensearch opensearch/opensearch --version=${OPEN_VERSION} --values=values-opensearch-worker.yaml > helm-opensearch-worker.yaml


opensearch: helm open-deployment

run-dump: 
	kubectl kustomize .

dump:  opensearch run-dump

run-apply:  
	kubectl apply -k .

apply:  opensearch run-apply

run-destroy:
	kubectl delete -k .

destroy:  opensearch run-destroy