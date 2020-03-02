# Cluster rabbitmq
Для отказоустойчивости  и маштабируемости RabbitMQ будем использовать Kubernetes.
В RabbitMQ  будет использоваться плагин RabbitMQ Peer Discovery Kubernetes. Для большой отказаустойчивости развернем кластер RabbitMQ как Stateful Set приложение с Persitent Volumes на кластере Ceph. 

### Создадим namespace
Для начала создадим namespace для кластера RabbitMQ:

 `kubectl create ns rabbitmq`
 
 ### Создадим сервисы
  `kubectl create -f services-rabbitmq.yaml`
 
Сервис rabbitmq-internal нужен для работы плагина Peer Discovery.
Для того что бы RabbitMQ был доступен из вне используем сервис NodePort. При обращении к любой ноде кластера по портам 30672 и 31672, нас будет направлять к подам кластера. Приложениями внутри Kubernetes кластер будет доступ по имени сервиса и порту 5672.

### Выдадим разрешения

`kubectl create -f rbac-rabbitmq.yaml`

### Storage class

Для большой отказоустойчивость мы будем использовать Persitent Volumes, которые создаются автоматически через Storage Class. В качестве хранилища я использовал Ceph. Использование отдельного хранилища позволяет в случае падения ноды развернуть под с с тем хранилище на другой, рабочей ноде.

### Развернуть ConfigMap

`kubectl create -f rabbitmq_configmap.yaml`

В конфигурации добавлены следующии параметры:
Добавляем плагины в качестве разрешенных:
`enabled_plugins: |
  [rabbitmq_management,rabbitmq_peer_discovery_k8s]`

Устанавливаем настройку для определения нод кластера по hostname
`cluster_formation.k8s.address_type = hostname`

Следующая настройку нужно устаналивать в зависимости от маштабируемости кластера, если кластер будет только увелчивать то параметр нужен устанавливать в true, если планиуется дескалировать, так как нагрузка уменьшилась, то нужно удалять лишнии поды и тогда надо установить настройку false

`cluster_formation.node_cleanup.only_log_warning = true`

Действия кластера при потери кворума:

`cluster_partition_handling = autoheal`

Определяем выбор мастера для новых очередей. При данной настройке мастером будет выбираться нода с наименьшим количеством очередей, таким образом очереди будут распределяться равномерно по нодам кластера.

`queue_master_locator=min-masters`

Определяем суффикс для FQDN пода
`cluster_formation.k8s.hostname_suffix = .rabbitmq-internal.rabbitmq.svc.cluster.local`

### Развернем StateFul Set для RabbitMQ

`kubectl create -f rabbitmq-statefulset.yaml`

В конфигурации устанавливаем количество реплик в значении 3 и указавыаем и ServiceName:
```
  serviceName: rabbitmq-internal
  replicas: 3
```
Указываем путь куда монтировать Persistent Volume:
```
- name: rabbitmq-data
    mountPath: /var/lib/rabbitmq/mnesia
```
Указываем переменные для FQDN имени.  Имя сервиса.
Содержимое Erlang Cookie должно быть одинаковым на всех нодах кластера, нужно прописать ваше собственное значение. Нода с отличающимся cookie не сможет войти в кластер.
```
          - name: RABBITMQ_NODENAME
            value: "rabbit@$(HOSTNAME).rabbitmq-internal.$(NAMESPACE).svc.cluster.local"
          - name: K8S_SERVICE_NAME
            value: "rabbitmq-internal"
          - name: RABBITMQ_ERLANG_COOKIE
            value: "mycookie"
 ```
 

В yaml конфиге добавляем настройку шаблонов для Persitent Volumes:
```
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: " rabbitmq-rbd"
      resources:
        requests:
          storage: 1Gi
```
