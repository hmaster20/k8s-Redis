# Redis Cluster

[Документация](https://ot-container-kit.github.io/redis-operator)

[Базовый репозиторий](https://github.com/OT-CONTAINER-KIT/redis-operator.git)

Основная ветка будет содержать описание.
Оператор и кластер будут в одноименных ветках

## Оператор

Оператор использует следующее пространство:
    namespace: redis-operator
Для наблюдения и управления, оператору указываем пространство "uat":
    watchNamespace: "uat"

## Кластер

При развертывании в разных пространствах, необходимо использовать RBAC для доступа оператора:
.\charts\redis-replication\templates\rbac.yaml

При развертывании чарта `redis-cluster` используется namespace: "uat"
Имя чарта скорее всего будет именем сервиса репликации, например:

```yaml
kind: Service
metadata:
  labels:
    app: redis-cluster
  name: redis-cluster
```

Поэтому для sentinel в файле `values-uat.yaml` нужно указывать следующим образом:

```yaml
redis-sentinel:
  redisSentinelConfig:
    redisReplicationName: redis-cluster
```

## Секреты

Для защиты данных необходимо задать значение для REDIS_PASSWORD


## Мониторинг

Для сбора метрик на сервисах должна быть аннотация. Обычно она включена по умолчанию.
При использовании Prometheus, для сбора метрик можно использовать additionalServiceMonitors.

```yaml
kind: Service
metadata:
  annotations:
    prometheus.io/port: '9121'
    prometheus.io/scrape: 'true'
  name: ucf-redis
```

## Примечание

При обновлении (удалении) объектов могут возникнуть проблемы.
Для решения проблемы зависания обнуляем файнализеры:

```shell
kubectl patch crd redisreplications.redis.redis.opstreelabs.in -n uat -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch crd redissentinels.redis.redis.opstreelabs.in -n uat -p '{"metadata":{"finalizers":[]}}' --type=merge
```
