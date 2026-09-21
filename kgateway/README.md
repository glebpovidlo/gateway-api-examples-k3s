# kgateway-gateway-api-example

Пример настройки kgateway Gateway API для k3s.

## Установка kgateway

docs: https://kgateway.dev/docs/envoy/latest/install/helm/

Устновка Gateway API CDR
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml
```

Установка helm chart для kgateway CDR
```bash
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.4.3 kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds 
```

Проверить применённые CRD можно так:
```
kubectl get crd | grep gateway.kgateway.dev
```

Установка helm chart для kgateway
```bash
helm upgrade -i -n kgateway-system kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
--version v2.4.3
```

Таким образом мы настраиваем "базу" для дальнейшей настройки маршрутизации и доступа в кластер

## Пример с приложением

docs: https://kgateway.dev/docs/envoy/latest/install/sample-app/

```bash
cd sample-app
```

1. Заускаем sample-приложение

Применяем манифест sample-приложения
```bash
kubectl apply -f httpbin.yaml
```

Проверяем его
```bash
kubectl -n httpbin get pods
```

2. Настройка Gateway

Создаём ресурс Gateway
```bash
kubectl apply -f gateway.yaml
```

Проверяем
```bash
kubectl get gateway http -n kgateway-system
```

Проверяем gateway proxy pod:
```bash
kubectl get po -n kgateway-system -l gateway.networking.k8s.io/gateway-name=http
```

В k3s автоматически создаётся LB, проверяем:
```bash
kubectl get svc -n kgateway-system
```

3. Настраиваем HTTPRoute для привязки приложения к Gateway

В [httproute.yaml](sample-app/httproute.yaml) заменяем `www.example.com` на IP-адрес нашего k3s сервера
Создаём HTTPRoute в том же namespace, что и sample-приложение:
```bash
kubectl apply -f httproute.yaml
```

Проверяем привязку HTTPRoute:
```bash
kubectl get -n httpbin httproute/httpbin -o yaml
```

4. Отправляем запрос

```bash
export INGRESS_GW_ADDRESS=$(kubectl get svc -n kgateway-system http -o=jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
```
```bash
echo $INGRESS_GW_ADDRESS
```
```bash
curl -i http://$INGRESS_GW_ADDRESS:8080/headers -H "host: $INGRESS_GW_ADDRESS:8080"    # Сработает, если вместо `www.example.com` в httproute.yaml указан ip-адрес
```
Можно ещё открыть в баузере и проверить там - `http://<ip_адрес>:8080/get`



5. Очистка

```bash
kubectl delete -f httpbin.yaml
```

```bash
kubectl delete httproute httpbin -n httpbin
```

```bash
kubectl delete gateway http -n kgateway-system
```

## Ресурсы, создаваемые Helm-чартом kgateway

ref: https://github.com/kgateway-dev/kgateway/tree/main/install/helm/kgateway

При выполнении команды `helm upgrade -i -n kgateway-system kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway --version v2.4.3` создаются следующие ресурсы.

### kgateway Control Plane

Чарт разворачивает **Deployment** `kgateway` в пространстве имён `kgateway-system`. Этот контроллер отвечает за трансляцию ресурсов Gateway API в конфигурацию xDS для прокси-серверов.

Контроллер предоставляет **Service** с именем `kgateway`, включающий следующие порты:

| Порт | Назначение |
|------|-----------|
| `9977` (grpc-xds) | Поток конфигурации xDS для Envoy |
| `9093` (health) | Эндпоинты проверки работоспособности `/readyz` и `/livez` |
| `9092` (metrics) | Эндпоинт Prometheus для сбора метрик |

**Примечание**: значение `replicaCount` по умолчанию — 1, контроллер работает в режиме leader-election.

### GatewayClass

Чарт создаёт кластерный ресурс Gateway API **GatewayClass**:

- `kgateway` — для Envoy-прокси, управляемый контроллером `kgateway.dev/kgateway`

### RBAC и ServiceAccount

Чарт создаёт **ServiceAccount** для контроллера и связанные с ним кластерные RBAC-ресурсы:

- **ClusterRole** `kgateway-role`
- **ClusterRoleBinding** `kgateway-role`

Кластерные права необходимы, поскольку контроллеру требуется создавать Deployment/Service/ServiceAccount в пространствах имён, отличных от `kgateway-system`, а также управлять кластерным ресурсом GatewayClass.

**Примечание**: для параметров RBAC можно указать `rbac.create=false`, чтобы пропустить создание этих ресурсов и привязать собственный ClusterRole.

### Что НЕ входит в этот чарт

**CRD Gateway API и CRD kgateway** не устанавливаются данным чартом. Они поставляются отдельным чартом `kgateway-crds`, который необходимо установить перед основным чартом.

Также чарт **не разворачивает прокси-серверы (Envoy data plane)**. Прокси создаются динамически контроллером при появлении ресурсов `Gateway` в кластере, а не в момент установки Helm-чарта. Конфигурация прокси определяется ресурсом `GatewayParameters`.
