# СТРУКТУРА ПРОЕКТА


```bash
|
├── backend                       - исходный код бэкэнда + Dockerfile + gitlab-ci.yml + манифесты
├── frontend                      - исходный код фронтэнда + Dockerfile + gitlab-ci.yml + манифесты
├── .gitlab-ci.yml                - родительский пайплайн для сборки и релиза образов бэкенда и фронтенда в Container Registry
```


# ЗАПУСК ПРИЛОЖЕНИЯ

## 1. Создайте кластер в Яндекс.Облаке

по инструкции https://cloud.yandex.ru/docs/managed-kubernetes/quickstart


## 2. Создайте в облаке хранилище с именем "momo-store-pictures"

загрузите туда картинки пельмененной

## 3. Настройте /.kube/config

- получите креды
```bash
yc managed-kubernetes cluster get-credentials --id <id_кластера> --external
```
- проверка доступности кластера:
```bash
kubectl cluster-info
```
- сделайте бэкап текущего ./kube/config:
```bash
cp ~/.kube/config ~/.kube/config.bak
```
- создайте манифест gitlab-admin-service-account.yaml:

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gitlab-admin
  namespace: kube-system
```
и примените его:
```bash
kubectl apply -f gitlab-admin-service-account.yaml
```
- получите endpoint: публичный ip адрес находится по пути Managed Service for Kubernetes/Кластеры/ваш_кластер -> обзор -> основное -> Публичный IPv4

- получите KUBE_TOKEN:
```bash
kubectl -n kube-system get secrets -o json | jq -r '.items[] | select(.metadata.name | startswith("gitlab-admin")) | .data.token' | base64 --decode
```
- сгенерируйте конфиг:
```bash
export KUBE_URL=https://<см. пункт выше>   # Важно перед IP указать https://
export KUBE_TOKEN=<см.пункт выше>
export KUBE_USERNAME=gitlab-admin
export KUBE_CLUSTER_NAME=<id_кластера> как в пункте_3

kubectl config set-cluster "$KUBE_CLUSTER_NAME" --server="$KUBE_URL" --insecure-skip-tls-verify=true
kubectl config set-credentials "$KUBE_USERNAME" --token="$KUBE_TOKEN"
kubectl config set-context default --cluster="$KUBE_CLUSTER_NAME" --user="$KUBE_USERNAME"
kubectl config use-context default
```


## 4. Установка Ingress-контроллера NGINX с менеджером для сертификатов Let's Encrypt

Чтобы с помощью Kubernetes создать Ingress-контроллер NGINX и защитить его сертификатом Let's Encrypt®, выполните следующие действия:

- установите NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
```

- установите менеджер сертификатов

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```


## 5. Узнайте IP-адрес Ingress-контроллера

значение в колонке EXTERNAL-IP:
```bash
kubectl get svc -n ingress-nginx
```


## 6. Создайте домен

(например, devops-zyoma.shop), укажите IP-адрес из п.5
сохраните этот субдомен в переменной SHOP_URL в Gitlab в настройках CI/CD


## 7. Создайте манифест acme-issuer.yaml

```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <емейл-адрес>
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
        ingress:
          class: nginx
```

примените его
```bash
kubectl apply -f acme-issuer.yaml
```


## 8. Для скачивания образов из Container Registry в k8s необходимо создать:
- секрет:
```bash
kubectl create secret docker-registry docker-config-secret --docker-server=gitlab.praktikum-services.ru:5050   --docker-username=<указать_свой_логин>   --docker-password=<указать_свой_пароль>
```

- сервис-аккаунт:
```bash
kubectl create serviceaccount my-serviceaccount
kubectl patch serviceaccount my-serviceaccount -p '{"imagePullSecrets": [{"name": "docker-config-secret"}]}' -n default 
```

## Готово!

Приложение доступно по адресу: https://devops-zimin.shop


# МОНИТОРИНГ

- скопируйте репозиторий отсюда: https://gitlab.praktikum-services.ru/root/monitoring-tools
- удостоверьтесь, что в описание приложения бэкенда пельменной (spec.template.annotations) добавлены аннотации:
  ```bash
prometheus.io/path: /metrics
prometheus.io/port: "8081"
prometheus.io/scrape: "true"
```

## 1. Установка Prometheus

- в скопированном репозитории в prometheus/templates создайте файл clusterRole.yaml:
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
    - nodes
    - nodes/proxy
    - nodes/metrics
    - services
    - endpoints
    - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: default
```
- выполните команду:
```bash
helm upgrade --install prometheus prometheus
```
- при необходимости настройте Ingress или заходите в веб-интерфейс с помощью port-forward:
```bash  
kubectl port-forward <имя_пода_prometheus> -n default 9090
```


## 2. Установка Grafana

- выполните команду:
```bash
helm upgrade --install grafana  grafana
```

- при необходимости настройте Ingress или заходите в веб-интерфейс с помощью port-forward:
```bash  
kubectl port-forward <имя_пода_grafana> -n default 3000
```

- импортируйте или настройте самостояльно нужный дашборд

Дашборды доступны тут: http://158.160.119.60:3000