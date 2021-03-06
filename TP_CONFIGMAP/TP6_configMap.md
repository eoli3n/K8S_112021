# TP configMap


- Utilisation d'un ConfigMap pour la configuration d'un reverse proxy

## 1. Context

- Dans cette mise en pratique nous allons voir l'utilisation de l'objet ConfigMap pour fournir un fichier de configuration à un reverse proxy très simple que nous allons baser sur *nginx*.

Nous allons configurer ce proxy de façon à ce que les requètes reçues sur le endpoint  soient forwardées sur un service nommé *php-service*, tournant également dans le cluster. Ce service expose le pod php

## 2. Deploiement 

### 2.1 : Deploiement php

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php
  labels:
    app: php
    projet: npm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php
  template:
    metadata:
      labels:
        app: php
    spec:
      initContainers:
      - name: side-car
        image: busybox
        volumeMounts:
        - name: php-volume
          mountPath: /php_code
        command:
        - wget
        - "-O"
        - "/php_code/index.php"
        - https://raw.githubusercontent.com/psable/php/main/myphp.php
      containers:
      - name: php
        image: phpdockerio/php73-fpm
        volumeMounts:
        - name: php-volume
          mountPath: /srv/http
      volumes:
        - name: php-volume
          emptyDir: {}
```

### 2.2 Deploiement service php-service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-service
  labels:
    projet: npm
    app: php
spec:
  type: ClusterIP
  selector:
    app: php
    projet: npm
  ports:
  - protocol: TCP
    port: 9000
    targetPort: 9000
```

### 2.3 Creation d'une configMap à partir du fichier nginx.conf

> https://kubernetes.io/docs/concepts/configuration/configmap/

> https://kubernetes.io/fr/docs/tasks/configure-pod-container/configure-pod-configmap/

- Approche impérative + output yaml
- Création du fichier yaml

```bash
$ kubectl create configmap nginx-config --from-file=nginx.conf --dry-run=client -o yaml > cm-nginx.yaml
```


## 2.4 Déclaration d'un deploiement pour nginx

    - utiliser la configMap précédente : utiliser la notion de volume pour présenter la configMap sous forme de fichier dans le POD

    - image : nginx:1.20-alpine

    ```yaml
    volumes:
      - name: config
        configMap:
          name: nginx-cm
          items:
          - key: config
            path: site.conf
    containers:
      volumeMounts:
      - name: config
        mountPath: /etc/nginx/conf.d
    ```

### 2.5 Creer un service de type nodePort pour acceder au serveur web depuis l'exterieur du cluster


### 2.6 Creation d'un secret pour l'injecter dans php

