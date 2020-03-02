# rabbitmq
Для отказоустойчивости  и маштабируемости RabbitMQ будем использовать Kubernetes.
В RabbitMQ  будет использоваться плагин RabbitMQ Peer Discovery Kubernetes. Для большой отказаустойчивости развернем кластер RabbitMQ как Stateful Set приложение с Persitent Volumes на кластере Ceph. 

### Создадим namespace
Для начала создадим namespace для кластера RabbitMQ:

 `kubectl create ns rabbitmq`
 
 ### Создадим сервисы
  `kubectl create ns rabbitmq services-rabbitmq.yaml`
 
Сервис rabbitmq-internal нужен для работы плагина Peer Discovery.
Для того что бы RabbitMQ был доступен из вне используем сервис NodePort. При обращении к любой ноде кластера по портам 30672 и 31672, нас будет направлять к подам кластера. Приложениями внутри Kubernetes кластер будет доступ по имени сервиса и порту 5672.
