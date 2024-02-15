# Процесс запуска Minikube с Jenkins и создания Jenkins Job для Nginx

## Описание

В данной инструкции рассматривается процесс развёртывания Minikube на вашем компьютере, развёртывания Jenkins в Minikube, а также процесс создания Jenkins Job для установки Helm чарта, задачей которого является установка последней версии Nginx, который затем будет доступен на ноде K8s по порту 32080.

## Шаги

### 1. Установка Minikube

Вы можете пропустить данный шаг, если на вашей машине уже установлен Minikube

#### Требования

- Убедитесь, что у вас установлен Docker (или какой-либо гипервизор на ваш выбор). Если это не так, то необходимо установить его.
- Убедитесь, что у вас установлен kubectl. Если это не так, то необходимо установить его:

  - Откройте терминал и загрузите последнюю версию kubectl с помощью команды:

    ```
    curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
    ```
     
  - Сделайте бинарный файл kubectl исполняемым:
    
    ```
    chmod +x ./kubectl
    ```
     
  - Переместите бинарный файл в директорию из переменной окружения PATH:
    
    ```
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```
     
  - Убедитесь, что установлена последняя версия:
    
    ```
    kubectl version --client
    ```

#### Установка

- Откройте терминал и выполните следующую команду, чтобы загрузить бинарный файл:
  
  ```
  curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
  ```
- Чтобы исполняемый файл Minikube был доступен из любой директории выполните следующие команды:
  
  ```
  sudo mkdir -p /usr/local/bin/
  sudo install minikube /usr/local/bin/
  ```
  
#### Запуск

- Чтобы убедиться, что Minikube был установлен корректно, выполните следующую команду, которая запускает локальный кластер Kubernetes:
  
> **Примечание**: Для использования опции `--vm-driver` с командой `minikube start` укажите имя установленного вами гипервизора в нижнем регистре в заполнителе `<driver_name>` команды ниже. **Например**: *docker*, *virtualbox*, *kvm2*. **Рекомендуемый выбор**: docker.

  ```
  minikube start --vm-driver=<driver_name>
  ```

- Проверьте статус Minikube с помощью команды:
  
  ```
  minikube status
  ```

### 2. Установка Jenkins

> **Примечание** все файлы следует создавать в директории проекта.

- Создайте отдельное пространство имён для Jenkins:
  
  ```
  kubectl create namespace devops-tools
  ```

- Создайте файл `serviceAccount.yaml` со следующим содержимым:
  
  ```yaml
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: jenkins-admin
  rules:
    - apiGroups: [""]
      resources: ["*"]
      verbs: ["*"]
  ---
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: jenkins-admin
    namespace: devops-tools
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: jenkins-admin
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: jenkins-admin
  subjects:
  - kind: ServiceAccount
    name: jenkins-admin
    namespace: devops-tools
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: deployer
  rules:
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["get", "list", "watch", "create", "delete", "patch", "update"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: deployer-binding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: deployer
  subjects:
  - kind: ServiceAccount
    name: jenkins-admin
    namespace: devops-tools
  ```
   
- Создайте serviceAccount в Kubernetes:
  
  ```
  kubectl apply -f serviceAccount.yaml
  ```
   
- Создайте файл `volume.yaml` со следующим содержимым:
  
  ```yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: local-storage
  provisioner: kubernetes.io/no-provisioner
  volumeBindingMode: WaitForFirstConsumer
  ---
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: jenkins-pv-volume
    labels:
      type: local
  spec:
    storageClassName: local-storage
    claimRef:
      name: jenkins-pv-claim
      namespace: devops-tools
    capacity:
      storage: 10Gi
    accessModes:
      - ReadWriteOnce
    local:
      path: /mnt
    nodeAffinity:
      required:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - minikube
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: jenkins-pv-claim
    namespace: devops-tools
  spec:
    storageClassName: local-storage
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 3Gi
  ```
   
- Создайте volume в Kubernetes:
  
  ```
  kubectl create -f volume.yaml
  ```

- Создайте файл `deployment.yaml` со следующим содержимым:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: jenkins
    namespace: devops-tools
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: jenkins-server
    template:
      metadata:
        labels:
          app: jenkins-server
      spec:
        securityContext:
              fsGroup: 1000
              runAsUser: 1000
        serviceAccountName: jenkins-admin
        containers:
          - name: jenkins
            image: jenkins/jenkins:lts
            resources:
              limits:
                memory: "2Gi"
                cpu: "1000m"
              requests:
                memory: "500Mi"
                cpu: "500m"
            ports:
              - name: httpport
                containerPort: 8080
              - name: jnlpport
                containerPort: 50000
            livenessProbe:
              httpGet:
                path: "/login"
                port: 8080
              initialDelaySeconds: 90
              periodSeconds: 10
              timeoutSeconds: 5
              failureThreshold: 5
            readinessProbe:
              httpGet:
                path: "/login"
                port: 8080
              initialDelaySeconds: 60
              periodSeconds: 10
              timeoutSeconds: 5
              failureThreshold: 3
            volumeMounts:
              - name: jenkins-data
                mountPath: /var/jenkins_home
        volumes:
          - name: jenkins-data
            persistentVolumeClaim:
                claimName: jenkins-pv-claim  
  ```

- Создайте deployment в Kubernetes:
  
  ```
  kubectl apply -f deployment.yaml
  ```

- Теперь можно проверить состояние deployment при помощи команды:

  ```
  kubectl get deployments -n devops-tools
  ``` 

- Создайте файл `service.yaml` со следующим содержимым:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: jenkins-service
    namespace: devops-tools
    annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/path:   /
        prometheus.io/port:   '8080'
  spec:
    selector:
      app: jenkins-server
    type: NodePort
    ports:
      - port: 8080
        targetPort: 8080
        nodePort: 32000
  ```

- Создайте service в Kubernetes:

  ```
  kubectl apply -f service.yaml
  ```

- Получите внешний IP-адрес и порт сервиса Jenkins:
  
  ```
  minikube service jenkins-service -n devops-tools --url
  ```

- Откройте Jenkins веб-интерфейс по адресу `http://<minikube-ip>:<port>`, используя полученный IP-адрес и порт вместо <minikube-ip> и <port> соответственно:
- Чтобы узнать `<jenkins-pod-name>` введите команду:
  
  ```
  kubectl get -n devops-tools Pods
  ```

- Введите команду для получения пароля для первоначального входа в Jenkins, заменив `<jenkins-pod-name>` на имя пода, полученное в ходе выполнения предыдущего шага:
  
  ```
  kubectl exec -it <jenkins-pod-name> -n devops-tools -- cat /var/jenkins_home/secrets/initialAdminPassword
  ```

  - Если это не сработало, то пароль можно также узнать, введя команду для получения логов, указанную ниже. В таком случае пароль будет находиться внизу
 
    ```
    kubectl logs <jenkins-pod-name> --namespace=devops-tools
    ```

### 3. Настройка Jenkins

- Скопируйте пароль и введите его на странице входа в Jenkins, после чего следуйте алгоритму действий на странице:
  
  - Выберите "Install suggested plugins", чтобы установить рекомендуемые плагины.
  - Дождитесь завершения установки плагинов. После этого создайте пользователя администратора Jenkins.
  - Войдите в систему в качестве администратора Jenkins.

### 4. Установка плагинов Jenkins
    
- Перейдите в раздел "Manage Jenkins" (Настроить Jenkins).
- В "System Configuration" выберите "Plugins" (Плагины).
- В разделе "Available plugins" (Доступные плагины) найдите нужный плагин и установите его. Вам потребуется следующие плагины:
  
  > **Примечание:** перезапускать Jenkins после установки плагина не обязательно, но можно.
  
  - Kubernetes Plugin: позволяет использовать Kubernetes для запуска сборок и деплоя в Jenkins.
  - Kubernetes CLI Plugin: позволяет использовать kubectl в пайплайне

### 5. Создание Jenkins Job

- Создайте новую Jinkins Job, выбрав тип Pipeline, введите желаемое Название.
- Пролистайте в самый низ и затем введите в соответствующее поле следующий скрипт:
> **ВНИМАНИЕ!!!** В stage 'Apply HELM Chart' в поле serverUrl вместо `<yourServerIP>` вам необходимо использовать IP вашей ноды кластера Kubernetes!
> **Примечание:** В случае необходимости вы можете изменить скрипт, скачав актуальную версию HELM и kubectl 

  ```bash
  pipeline {
      agent any
      stages{
          stage('Install HELM') {
              steps {
                  sh 'pwd'
                  sh 'curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz'
                  sh 'tar -zxvf helm.tar.gz '
                  sh 'chmod +x linux-amd64/helm'
                  sh './linux-amd64/helm version'
              }
          }
          stage('Pull Repo') {
              steps {
                  git 'https://github.com/DaniilBo/test-tasks.git'
                  sh 'pwd'
                  sh 'ls -l'
              }
          }
          stage('Install Kubectl'){
              steps {
                  sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
                  sh 'chmod u+x ./kubectl'
              }
          }
          stage('Apply HELM Chart') {
              steps {
                  withKubeConfig([serverUrl: '<yourServerIP>:8443']) {
                      sh './linux-amd64/helm upgrade --install nginx-chart /var/jenkins_home/workspace/test-task/nginx-chart -n devops-tools'
                      sh './kubectl get pods -n devops-tools'
                  }
              } 
          } 
      }
  }          
  ```

### 6. Проверка доступности Nginx

Чтобы проверить доступность nginx по требуемому порту (в данном случае: 32080), нужно:

- Узнать IP адрес нода, используя следующую команду:

  ```
  minikube ip
  ```

- Далее можно использовать один из способов на выбор:

  - Ввести команду ниже, заменив `<minikube-ip>` на IP адреc нода, полученный в ходе выполнения предыдущего шага, а `<port>` на порт, требуемый nginx (а данном случае - 32080):
    
    ```
    curl <minikube-ip>:<port>
    ```
    
  - Зайти на страницу при помощи браузера по следующему адресу `http://<minikube-ip>:<port>`, заменив `<minikube-ip>` на IP адреc нода, полученный в ходе выполнения предыдущего шага, а <port> на порт, требуемый nginx (а данном случае - 32080):
