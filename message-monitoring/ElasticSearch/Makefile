ELASTIC_VERSION := 0.8.0

helm:
	helm repo add elastic https://helm.elastic.co
	helm repo update elastic


elastic-deployment:
	helm template elasticsearch elastic/eck-elasticsearch --version=${ELASTIC_VERSION} --values=values-elasticsearch.yaml > helm-elasticsearch.yaml
	vi elasticsearch/helm-elasticsearch.yaml -c ":%s/enterprise/basic/g" -c ":wq"

elasticsearch: helm elasticsearch-deployment

run-dump: 
	kubectl kustomize .

dump:  elasticsearch run-dump

run-apply:  
	kubectl apply -k .

apply:  elasticsearch run-apply

run-destroy:
	kubectl delete -k .

destroy:  elasticsearch run-destroy