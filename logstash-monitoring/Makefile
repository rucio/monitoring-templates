LOGSTASH_VERSION := 0.10.0

helm:
	# helm repo add elastic https://helm.elastic.co
	# helm repo update elastic


logstash-deployment:
	helm template logstash elastic/eck-logstash --version=${LOGSTASH_VERSION} --values=logstash/values-logstash.yaml > logstash/helm-logstash.yaml

logstash: helm logstash-deployment

run-dump: 
	kubectl kustomize .

dump:  logstash run-dump

run-apply:  
	kubectl apply -k .

apply:  logstash run-apply

run-destroy:
	kubectl delete -k .

destroy:  logstash run-destroy