# Лабораторная работа №9: Оптимизация образов контейнеров

## Задание: Сравнить различные методы оптимизации образов

### Опсиание вывполнения работы

1. Создаем репозиторий ```containers09``` и копируем его на компьютер.

2. В директории ```containers09``` создаем директорию ```./site```. В данной директории будут следующие файлы: index.html, style.css, script.js.

3. В корневой папке создаем ```Dockerfile.raw``` со следующим содержимым:

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y

# install nginx
RUN apt-get install -y nginx

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Создаем образ и собираем его с именем ```mynginx:raw```, используем команду:

 ```bash
docker image build -t mynginx:raw -f Dockerfile.raw .
```

![Img-1](https://imgur.com/WvBDGMK.png)

Это мы видим на странице в барузере:

![Img-2](https://imgur.com/l1Sy2Ks.png)

4. Создаем файл ```Dockerfile.clean```, который удаляет временные файлы и неиспользуемые зависимости. У этого файла следующее содержимое:

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y

# install nginx
RUN apt-get install -y nginx

# remove apt cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Создаем образ и собираем его с именем ```mynginx:clean```, используем команду:

```bash
docker image build -t mynginx:clean -f Dockerfile.clean .
```

![Img-3](https://imgur.com/CtznhFV.png)

А также проверяем список  и размеры образов при помощи команды:

```bash
docker image list
```

![Img-4](https://imgur.com/ehqSlwg.png)

5. Создаем файл ```Dockerfile.few``` для уменьшения колличества слоев со следующем содержимым:

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Собераем образ с именем ```mynginx:few``` и проверяем его размер:

```bash
docker image build -t mynginx:few -f Dockerfile.few .
```

![Img-5](https://imgur.com/PlEVT9L.png)
![Img-6](https://imgur.com/6n2he4x.png)

6. Меняем базовый образ на ```alpine``` и пересобераем образ:

```dockerfile
# create from alpine image
FROM alpine:latest

# update system
RUN apk update && apk upgrade

# install nginx
RUN apk add nginx

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Собераем образ с именем ```mynginx:alpine``` и проверяем его размер:

![Img-7](https://imgur.com/rjl6Q64.png)
![Img-8](https://imgur.com/cbFFJzi.png)

7. Перепаковываем обарз ```mynginx:raw``` в ```mynginx:repack:```

```bash
docker container create --name mynginx mynginx:raw
docker commit mynginx mynginx:repack
docker container rm mynginx
```

8. Создаем образ ```mynginx:min``` с использованием всех методов:

```dockerfile
# create from alpine image
FROM alpine:latest

# update system, install nginx and clean
RUN apk update && apk upgrade && \
    apk add nginx && \
    rm -rf /var/cache/apk/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Собераем образ с именем ```mynginx:minx``` и проверьте его размер. Перепакуйте образ ```mynginx:minx``` в ```mynginx:min:```

![Img-9](https://imgur.com/6t7cq3N.png)

Далее используем следующие команды и выводим таблицу всех образов

```bash
docker image build -t mynginx:minx -f Dockerfile.min .
docker container create --name mynginx mynginx:minx
docker container export mynginx | docker image import - myngin:min
docker container rm mynginx
docker image list
```

![Img-10](https://imgur.com/24cskVr.png)

#### Ответы на вопросы

1. Какой метод оптимизации образов вы считаете наиболее эффективным?
Наиболее эффективным методом оптимизации образов в данном случае является использование минимального базового образа, например alpine. Этот метод значительно снижает размер образа, так как Alpine — это очень легкий образ, в котором включены только базовые пакеты для работы с приложением. По сравнению с более тяжелыми образами, такими как ubuntu, он позволяет уменьшить общий размер без потери функциональности.
Кроме того, использование всех методов одновременно (минимальный образ, очистка временных файлов, объединение слоев) также дает хорошие результаты, особенно когда нужно уменьшить размер контейнера без значительных изменений в его функциональности.

2. Почему очистка кэша пакетов в отдельном слое не уменьшает размер образа?
Очистка кэша пакетов в отдельном слое **не уменьшает размер образа**, потому что Docker сохраняет каждый слой как отдельный объект. Когда мы выполняем операцию очистки кэша в отдельном слое, сам слой, содержащий кэш, все равно существует в образе, даже если его содержимое было удалено.
Для эффективной оптимизации важно выполнять очистку в том же слое, где устанавливаются пакеты, чтобы избежать создания дополнительного слоя, содержащего неиспользуемые файлы. В противном случае Docker будет хранить этот промежуточный слой, что увеличивает размер финального образа.

3. Что такое перепаковка образа?

**Перепаковка образа** — это процесс, при котором контейнер создается из существующего образа, затем его файловая система экспортируется и сохраняется в новый образ. Это позволяет удалить метаданные и слои, которые Docker использует для хранения истории изменений контейнера, и создать более "чистую" версию образа.

*Перепаковка выполняется через команды:*

- Создание контейнера: ```docker container create --name mynginx mynginx:raw```
- Экспорт файловой системы контейнера: ```docker export mynginx```
- Импорт файловой системы в новый образ: ```docker image import - mynginx:repack```

Таким образом, перепаковка может уменьшить размер образа, удалив ненужные слои, метаданные и улучшив его структуру. Это полезно, если требуется создать более компактную и оптимизированную версию образа.

#### Вывод

В ходе работы были исследованы методы оптимизации Docker-образов, включая использование минимальных базовых образов, удаление неиспользуемых зависимостей, уменьшение количества слоев и перепаковку образа. Наиболее эффективным методом оказался использование минимального базового образа, который значительно снижает размер образа. Кроме того, комбинация методов, таких как очистка временных файлов и уменьшение слоев, позволяет дополнительно сократить размер контейнеров, улучшая их эффективность.
