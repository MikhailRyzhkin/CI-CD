# CI-CD

Спринт 2

ЗАДАЧА

```
1. Клонируем репозиторий, собираем его на сервере srv.
Исходники простого приложения можно взять здесь. Это простое приложение на Django с уже написанным Dockerfile. 
Приложение работает с PostgreSQL, в самом репозитории уже есть реализация docker-compose — её можно брать за 
референс при написании Helm-чарта. Необходимо склонировать репозиторий выше к себе в Git и настроить пайплайн 
с этапом сборки образа и отправки его в любой docker registry. Для пайплайнов можно использовать GitLab, 
Jenkins или GitHub Actions — кому что нравится. Рекомендуем GitLab.

2. Описываем приложение в Helm-чарт.
Описываем приложение в виде конфигов в Helm-чарте. По сути, там только два контейнера — с базой и приложением, 
так что ничего сложного в этом нет. Стоит хранить данные в БД с помощью PVC в Kubernetes.

3. Описываем стадию деплоя в Helm.
Настраиваем деплой стадию пайплайна. Применяем Helm-чарт в наш кластер. Нужно сделать так, чтобы наше приложение 
разворачивалось после сборки в Kubernetes и было доступно по бесплатному домену или на IP-адресе с выбранным портом.
Для деплоя должен использоваться свежесобранный образ. По возможности нужно реализовать сборку из тегов в Git, где 
тег репозитория в Git будет равен тегу собираемого образа. Чтобы создание такого тега запускало пайплайн на сборку 
образа c таким именем hub.docker.com/skillfactory/testapp:2.0.3.
```

РЕШЕНИЕ

Подзадача 1: Клонируем репозиторий, собираем его на сервере srv.
  - Клонируем указанный репозиторий тестового приложения. У нас это папка: django-pg-docker-tutorial
  - Дорабатываем приложение - выносим секретные данные по подключению к БД в папку "private_data" и вносим её 
  в гитигнор. Для примера, как выглядит этот файл - private.var.example. Исправляем ошибки.
  - Делаем исполняемыми скрипты в папке scripts:
  ```
  chmod +x test_deploy_app.sh
  chmod +x test_destroy_app.sh
  ```
  - тестово разворачиваем приложение на ноде srv в docker для дебагинга:
  ```
  cd /django-pg-docker-tutorial-clone && /opt/CI-CD/scripts/test_deploy_app.sh
  ```
  Видим удачное тестовое разворачивание:

![test app django-0](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/d8f85c68-7b99-42e7-a6c6-05bd13e3f465)


![test app django](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/fdd3b6f4-568d-4e72-b841-09f4b8ec85ee)

  - Так как приложение работает, то удаляем тестовое разворачивание приложения в docker:
  ```
  cd /django-pg-docker-tutorial-clone && /opt/CI-CD/scripts/test_destroy_app.sh
  ```
  - В рамках подзадачи в роли Docker registry будем использовать Dockerhub. Логинимся в Dockerhub.
  ```
  docker login -u mikhailrizhkin1
  ```
  - Cоздаёми там репозиторий для хранения артифакта сборки. В нашем случае это: mikhailrizhkin1/diplom
![docker-hub_diplom](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/485ff101-6c72-4ba2-84f1-f8f2bbead76b)
  
  - В качестве CI/CD будем использовать Gitlab-CI
  - Создаём репозиторий в gitlab.com под проект: https://gitlab.com/Mikhail_Ryz/diplom_skillfactory_ci-cd
  - Создаём в нём гитлаб-раннер по пути Settings-CI/CD-Runners и отключаем шарэд раннеры
![Настройка Gitlab-Runner](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/391f0a31-d137-4834-88c6-a3379d081fa5)

  - На сервере srv, настраиваем Gitlab-Runner:
  ```
  gitlab-runner register --url https://gitlab.com --token <your_token>
  ```
  И если видим его зелённым цветом, то всё в порядке.
![Настройка Gitlab-Runner-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/0cc6064d-5125-46e4-9f09-f94faaf1bec5)

  - Вносим пользователей в разрешение на docker
  ```
  usermod -aG docker ubuntu
  usermod -aG docker root
  usermod -aG docker gitlab-runner
  ```
  - Создаём нужные нам переменные для хранения приватных данных и другой информации по пути Settings-CI/CD-Variables:
![Variables](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/eb7b0769-5cc8-42a2-a5a3-14521f0cf3ab)

  - Описываем pipeline в стандартном .gitlab-ci.yml файле состоящий из двух стадий - сборки приложения из докерфайла и выкатка - деплой приложения. Запускаем пробно pipeline, который собирает
    проект и пушит его как артефакт в докер хаб. Результат тестовой работы pipeline:
  ![Результат pipeline](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/27ad0be3-60a1-4c51-a17f-6b218d48c427)

  ![Результат pipeline-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/024405df-6d53-4f58-bd31-1d67f4014125)

  - Для разворачивания приложения в  кластера kubernetes пишем манифесты в каталоге "Kube-manifests".
  - Секретные данные кодируем и помещаем в манифест "credentials.yaml":
  ```
  echo -n 'what_we_want_to_encode' | base64
  ```
  - Для приложения создадим отдельный нэймспейс, куда будем его диплоить:
  ```
  kubectl create namespace diplom
  ```
  - Переходим в каталог с манифестами и деплоим приложение в ранее развёрнутый K8S кластер:
  ```
  kubectl apply -f . -n diplom 
  ```
![Deploy app-0](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/eb172890-d8ef-4289-bacd-be348ae88824)

![Deploy app-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/75b7cd26-d7a1-4e30-9c91-2a2c3ee82d73)

![Deploy app-2](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/5fb7889b-28d7-4a6a-bbc9-9b2483fc043b)

![Deploy app-3](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/de6ef047-d411-4f60-838e-6e83a9f58fb1)


  - Для удаления приложения с кластера переходим в каталог с манифестами:
  ```
  kubectl delete -f . -n diplom 
  ```


Подзадача 2: Описываем приложение в Helm-чарт.
  - На базе нашего приложения м манифестов создаём Helm chart. Переходим в папку с манифестами и формируем свой чарт:
  ```
  helm create app-dpdt
  tree app-dpdt
  ```
![helm](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/6102a2ee-c28e-4b17-a029-0b630eb97326)
  - Редактируем созданную helm-заглушку. В первую очередь манифесты, которые берём с предыдущих шагов
  - Развернём наш Helm chart в кластере k8s. Переходим в папку с c чартом и запускаем установку с учетом данных в credentials.yaml:
  ```
  helm upgrade --install -n diplom --values templates/credentials.yaml --set service.type=NodePort app-dpdt .
  ```
  
Подзадача 3: Описываем стадию деплоя в Helm
  - Архивируем созданный helm-chart из папки выше чарта:
  ```
   helm package ./app-dpdt
  ```
  - Копируем и пушим созданный архив и сам чарт в репозиторий GitLab c CI/CD
  - Разрабатываем второй стейдж ci для развёртывания приложения в кластере k8s новой версии на основании изменения тэга
  То есть у нас тригер будет - тэг.

![Helm-deploy-2](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/2bb8f713-4330-4f46-acb8-b2d9c285eeec)

![Helm-deploy-3](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/eb34faf5-1f3f-43d2-b559-fbc37dff0d29)

![Helm-deploy](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/4b30cc55-2163-41a7-b5df-4c34b01cc7db)

![Helm-deploy-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/f800619a-da0a-4771-9b7e-d54ab7c4bdbb)

![pipeline](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/30afd39c-881a-4d83-9824-8a226ee37246)

![pipeline-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/6d17715c-d79f-4179-bdfd-66bdfdc1ae47)

Приложение доступно по любому из двух внешних IP (worker или master):
  ```
  http://158.160.75.53:30036/
  ```
или
  ```
  http://158.160.18.130:30036/
  ```






