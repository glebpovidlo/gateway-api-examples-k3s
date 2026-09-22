# traefik-gateway-api-example

Пример настройки Traefik Gateway API для k3s.

Для установки используется официальный Helm Chart установки Traefik в кластер.
В моём случае я использую namespace с именем `traefik-system`

## Подготовка

docs: https://doc.traefik.io/traefik/setup/kubernetes/

Добавление репозитория:
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Создание namespace:
```
kubectl create namespace traefik-system
```

## Применение Helm Chart

Установка
```
helm upgrade traefik traefik/traefik -i \
  --namespace traefik-system \
  --values values.yaml
```

Обновление конфигурации
```
helm upgrade traefik traefik/traefik \
  --namespace traefik-system \
  --values values.yaml
```

Удаление Helm Chart
```
helm uninstall traefik --namespace traefik-system
```

## Пример с приложением

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

2. Настраиваем HTTPRoute для привязки приложения к Gateway

Создаём HTTPRoute в том же namespace, что и sample-приложение:
```bash
kubectl apply -f httproute.yaml
```

Проверяем привязку HTTPRoute:
```bash
kubectl get -n httpbin httproute/httpbin -o yaml
```

3. Отправляем запрос

```bash
export INGRESS_GW_ADDRESS=$(kubectl get svc traefik -n traefik-system -o=jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
```
```bash
echo $INGRESS_GW_ADDRESS
```
```bash
curl -i http://$INGRESS_GW_ADDRESS:8080/get
```
Можно ещё открыть в баузере и проверить там - `http://<ip_адрес>:8080/get`

4. Очистка

```bash
kubectl delete httproute httpbin -n httpbin
```

```bash
kubectl delete -f httpbin.yaml
```

## Kubernetes-ресурсы, создаваемые Helm Chart

### Общая картина

Чарт **отключает классический Ingress** и переводит Traefik в режим **Gateway API**. Дополнительно:

- открывается **дашборд** через отдельный EntryPoint `traefik`,
- включаются **метрики Prometheus** и **access-логи**,
- порт `websecure` (HTTPS) **не публикуется наружу**, но может оставаться в конфигурации по умолчанию.

### Deployment / DaemonSet (Traefik)

- Запускается **под Traefik** с двумя контейнерами (сам Traefik + init-контейнер для настройки, если требуется).
- **EntryPoints** внутри пода (статическая конфигурация):

| Имя | Порт контейнера | Назначение |
|---|---|---|
| `web` | `8080` | Приём HTTP-трафика от Gateway |
| `traefik` | `8100` | API + дашборд |
| `metrics` | `9100` | Prometheus-метрики |
| `websecure` | (по умолчанию) | не публикуется (`expose.default: false`) |

- Включён **дашборд**, но **insecure API выключен** (`api.insecure: false`) — значит дашборд доступен только через EntryPoint `traefik` (`8100`).

### Service (LoadBalancer / NodePort)

Kubernetes-сервис публикует порты:

| Имя порта | `port` (контейнер) | `exposedPort` (Service) | `nodePort` | Примечание |
|---|---|---|---|---|
| `web` | 8080 | 8080 | 30080 | Основной HTTP |
| `traefik` | 8100 | 8100 | 30100 | API / дашборд |
| `metrics` | 9100 | 9100 | 31100 | Метрики Prometheus |
| `websecure` | — | — | — | Не публикуется |

> Тип сервиса зависит от `service.type` (по умолчанию в чарте — `LoadBalancer`). Если `LoadBalancer` — k3s выдаст внешний IP, а `nodePort` будет выделен на всех нодах.

### ServiceAccount, ClusterRole, ClusterRoleBinding

- Создаются для Traefik, чтобы он мог читать ресурсы Gateway API и IngressRoute.

### CRD (если не установлены ранее)

- `IngressRoute`, `Middleware`, `TLSOption` и др. — CRD Traefik.

> Если включён провайдер `kubernetesGateway` — используются CRD **Gateway API** (`GatewayClass`, `Gateway`, `HTTPRoute`), которые должны быть установлены в кластере отдельно (обычно вместе с Gateway API CRDs).

### GatewayClass

- Создаётся ресурс **`GatewayClass`** с контроллером Traefik (`traefik.io/gateway-controller`). Это «класс» для будущих Gateway.

### Gateway (Gateway API)

- Создаётся ресурс **`Gateway`** с двумя слушателями (listeners):

| Listener | Порт (ссылка на EntryPoint) | Protocol | NamespacePolicy |
|---|---|---|---|
| `web` | 8080 | HTTP | `All` |
| `traefik` | 8100 | HTTP | `All` |

> Оба listener ссылаются на существующие EntryPoints из блока `ports` (`web` → 8080, `traefik` → 8100). Связь идёт по номеру порта контейнера.

### IngressRoute (для дашборда)

Создаётся ресурс **`IngressRoute`** (CRD Traefik), который публикует дашборд:

- **matchRule**: `Host(\`hostname\`)`
- **entryPoints**: `traefik` (порт 8100)
- **Сервис**: внутренний API Traefik (dashboard)

Таким образом, дашборд будет доступен по адресу: `http://hostname:8100/` (если `hostname` — IP/Домен, назначенный сервису/ноде).

### ConfigMap / Secret

- **ConfigMap** со статической конфигурацией Traefik (entryPoints, providers, metrics, logs).
- Возможно, **Secret** для TLS (если используется) — в данной конфигурации TLS не настроен.

### RBAC для провайдеров

- `kubernetesGateway: enabled: true` → Traefik получает права на чтение `Gateway`, `HTTPRoute`, `GatewayClass`.
- `kubernetesIngress: enabled: false` → классические Ingress не обрабатываются.

### Что НЕ создаётся

| Ресурс | Причина |
|---|---|
| `IngressClass` | `ingressClass.enabled: false` |
| Классические `Ingress` | `kubernetesIngress.enabled: false` |
| HTTPS-listener (`websecure`) | `expose.default: false` |
| Внешний доступ к дашборду через Ingress | Дашборд идёт через `IngressRoute` + EntryPoint `traefik` |

### Схема потоков трафика

```
Пользователь → Service (LB, exposedPort 8080) → под Traefik: EntryPoint web (8080)
                                                    ↓
                                            Gateway API (listener web)
                                                    ↓
                                            HTTPRoute (создаётся пользователем)
                                                    ↓
                                            Сервис приложения

Админ → Service (LB, exposedPort 8100) → под Traefik: EntryPoint traefik (8100)
                                                    ↓
                                            IngressRoute dashboard
                                                    ↓
                                            Дашборд Traefik

Prometheus → Service (LB, exposedPort 9100) → под Traefik: EntryPoint metrics (9100)
```

### Замечания

1. **Gateway API CRDs** должны быть установлены в кластере заранее (например, `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/...`). Чарт Traefik их не устанавливает.

2. **`metrics.prometheus.enabled: true`** открывает `/metrics` на EntryPoint `metrics` (9100), который опубликован через сервис — это позволяет Prometheus скрейпить метрики.

3. **`expose.default: true`** обязателен для портов `web`, `traefik`, `metrics` — без него порт не попадёт в список EntryPoints сервиса LB, и listener Gateway не сможет с ним связаться.

4. **Хост для дашборда** (`matchRule: Host(\`hostname\`)`) нужно заменить на реальный IP или домен, назначенный сервису/ноде, иначе дашборд будет недоступен.

### Краткая сводка создаваемых объектов

| Ресурс | Имя / параметр | Назначение |
|---|---|---|
| Deployment | `traefik` | Под с Traefik |
| Service | `traefik` (LB/NodePort) | Публикация портов 8080, 8100, 9100 |
| GatewayClass | `traefik` | Класс Gateway для Traefik |
| Gateway | `traefik-gateway` (или по имени релиза) | Listeners `web` (8080), `traefik` (8100) |
| IngressRoute | dashboard | Публикация дашборда по Host `hostname` через EntryPoint `traefik` |
| ConfigMap | traefik | Статическая конфигурация |
| ServiceAccount + RBAC | traefik | Доступ к Gateway API |
| CRD (Traefik) | IngressRoute и др. | Если не установлены |
| CRD (Gateway API) | GatewayClass, Gateway, HTTPRoute | Требуются отдельно |
