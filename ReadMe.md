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

### CSI

Для защиты данных необходимо задать значение для REDIS_PASSWORD
В примере используется kind: SecretProviderClass и provider: azure, секрет читается из Azure Key Vault

Как альтернатива,  если хотим использовать CSI, можно создать два файла:

redis-k8s-secret.yaml

```yaml
apiVersion: v1
kind: Secret
meta
  name: redis-secret
  namespace: {{ .Release.Namespace }}
type: Opaque
data:
  UAT_REDIS_PASSWORD: JUtKSyY5YXNEbGpTMzQ=
```

redis-secret-provider.yaml

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
meta
  name: redis-secret
  namespace: {{ .Release.Namespace }}
spec:
  provider: kubernetes
  parameters:
    objects: |
      - objectName: "UAT_REDIS_PASSWORD"
        key: "UAT_REDIS_PASSWORD"
        objectAlias: "UAT_REDIS_PASSWORD"
```

### no CSI

Из секции storageSpec убираем csi

```yaml
storageSpec:
  volumeMount:
    volume:
      - name: redis-secret
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: redis-secret
    mountPath:
      - mountPath: "/var/uat-redis"
        name: redis-secret
```

Будет вот так:

```yaml
storageSpec:
  volumeMount:
    volume:
      - name: redis-secret
        secret:
          secretName: redis-secret
    mountPath:
      - mountPath: "/var/uat-redis"
        name: redis-secret
```

Секрет будет получен через envFrom и volumeMount

## Мониторинг

Для сбора метрик на сервисах должна быть аннотация. Обычно она включена по умолчанию.
При использовании Prometheus, для сбора метрик можно использовать additionalServiceMonitors.

```yaml
kind: Service
metadata:
  annotations:
    prometheus.io/port: '9121'
    prometheus.io/scrape: 'true'
```

## Примечание

В секрете используем data, потому что stringData несовместим с SecretProviderClass в некоторых реализациях CSI.
Пароль хешируется командой `echo -n "%KJK&9asDljS34" | base64`

При обновлении (удалении) объектов могут возникнуть проблемы.
Для решения проблемы зависания обнуляем файнализеры:

```shell
kubectl patch crd redisreplications.redis.redis.opstreelabs.in -n uat -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch crd redissentinels.redis.redis.opstreelabs.in -n uat -p '{"metadata":{"finalizers":[]}}' --type=merge
```
