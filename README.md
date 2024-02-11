# Процесс запуска Minikube с Jenkins и создания Jenkins Job для Nginx

## Описание

В данной инструкции рассматривается процесс развёртывания Minikube на вашем компьютере, развёртывания Jenkins в Minikube, а также процесс создания Jenkins Job для установки Helm чарта, задачей которого является установка последней версии Nginx, который затем будет доступен на ноде K8s по порту 32080.

## Шаги

### 1. Установка Minikube

Вы можете пропустить данный шаг, если на вашей машине уже установлен Minikube

#### Требования

- Убедитесь, что у вас установлен Docker (или какой-либо гипервизор на ваш выбор). Если это не так, то необходимо установить его.
- Убедитесь, что у вас установлен kubectl. Если это не так, то необходимо установить его:

  1. Откройте терминал и загрузите последнюю версию kubectl с помощью команды:

     ```
     curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
     ```
     
  2. Сделайте бинарный файл kubectl исполняемым:
     ```
     chmod +x ./kubectl
     ```
     
  3. Переместите бинарный файл в директорию из переменной окружения PATH:
     ```
     sudo mv ./kubectl /usr/local/bin/kubectl
     ```
     
  4. Убедитесь, что установлена последняя версия:
     ```
     kubectl version --client

     ```

#### Установка

- Откройте терминал и выполните следующую команду, чтобы загрузить двоичный файл:
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
  
> **Примечание**: Для использования опции `--vm-driver` с командой `minikube start` укажите имя установленного вами гипервизора в нижнем регистре в заполнителе `<driver_name>` команды ниже. **Например**: *docker*, *virtualbox*, *kvm2*.
```
minikube start --vm-driver=<driver_name>
```

- Проверьте статус Minikube с помощью команды:
```
minikube status
```

### 2. Установка Jenkins

- Создайте файл `jenkins-deployment.yaml` со следующим содержимым:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: jenkins
     labels:
       app: jenkins
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: jenkins
     template:
       metadata:
         labels:
           app: jenkins
       spec:
         containers:
           - name: jenkins
             image: jenkins/jenkins:lts
             ports:
               - containerPort: 8080
               - containerPort: 50000
   ```
   
- Создайте Deployment в Kubernetes:
   ```
   kubectl apply -f jenkins-deployment.yaml
   ```
   
- Создайте файл `jenkins-service.yaml` со следующим содержимым:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: jenkins-service
   spec:
     type: NodePort
     ports:
       - name: http
         port: 8080
         targetPort: 8080
         nodePort: 30000
       - name: jnlp
         port: 50000
         targetPort: 50000
     selector:
       app: jenkins
   ```
   
- Создайте Service в Kubernetes:
   ```
   kubectl apply -f jenkins-service.yaml
   ```

- Получите внешний IP-адрес сервиса Jenkins:
   ```
   minikube service jenkins-service --url
   ```

- Откройте Jenkins веб-интерфейс, используя полученный IP-адрес с портом 30000:
   ```
   http://<minikube-ip>:30000
   ```

- Введите команду для получения пароля для входа в Jenkins:
   ```
   kubectl exec -it <jenkins-pod-name> -- cat /var/jenkins_home/secrets/initialAdminPassword
   ```

- Скопируйте пароль и введите его на странице входа в Jenkins, после чего следуйте алгоритму действий на странице.

Теперь Jenkins успешно развернут в Minikube и готов к использованию.
