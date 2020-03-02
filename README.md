# rabbitmq
Для отказоустойчивости  и маштабируемости RabbitMQ будем использовать Kubernetes.
В RabbitMQ  будет использоваться плагин RabbitMQ Peer Discovery Kubernetes. Для большой отказаустойчивости развернем кластер RabbitMQ как Stateful Set приложение с Persitent Volumes на кластере Ceph. 
