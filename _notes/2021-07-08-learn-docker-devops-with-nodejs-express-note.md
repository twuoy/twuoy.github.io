---
title: Learn Docker - DevOps with Node.js & Express 筆記
tags: Docker DevOps
cover: /assets/images/2021-07-08-learn-docker-devops-with-nodejs-express-note-ian-taylor-jOqJbvo1P9g-unsplash.jpg
article_header:
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/2021-07-08-learn-docker-devops-with-nodejs-express-note-ian-taylor-jOqJbvo1P9g-unsplash.jpg

---

> *Learn Docker - DevOps with Node.js & Express 筆記*

<!--more-->
---

Photo by <a href="https://unsplash.com/@carrier_lost?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Ian Taylor</a> on <a href="https://unsplash.com/s/photos/container?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>.

# Docker image layers & caching

Timeline: [0:10:34](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=634s)

Dockerfile 裡的每個步驟都是一個 layer ，layer 是可以被 cache 的，但每個 layer 會 based on 上面所有 layer，所以只要上面步驟有更動，那其下步驟就會重新 build。

### 優化 build image 效能的方法
> Install package before copy file to avoid reinstall the same packages due to changes in other file.

- 因為每次更版重新 build image 大多數情況是 code 有改動，而不需要更新 package (e.g. requirements.txt 沒更動)，所以將 `COPY ./requirements.txt . RUN pip install -r requirements.txt` 放在 copy project 前執行才能享受到 cache 的好處。

# Docker networking opening ports

Timeline: [0:20:26](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=634s)

## Network Setting Inside Dockerfile and Docker Compose File
- **By default, docker will isolate the container if we don't open up any ports.**

## EXPOSE

- **Does nothing, more for documentation purposes.**
    - If others use these files, they knows what ports need to be opened.

# Syncing source code with bind mounts

Timeline: [0:31:46](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=1906s)

## Docker volume

### Bind mount

- Sync a folder/file system on host machine to a folder within container.
- Usually used in development, because we don't want to change code in production.
- When using docker run, the path can't use `dot syntax`.

**Create bind mount volume using docker run**

```bash
docker run -v path_to_folder_on_host:path_to_folder_on_container
```

- host machine folder 的內容會**覆寫/取代** container 的內容。
    - 要避免 container 指定 folder 下的內容全被取代，可以藉由再指定更特定(specific)的 container path 來 override。
        - `docker run -v path_to_folder_on_host:path_to_folder_on_container/ -v /path_to_folder_on_container/{more_specific_path}`

### Anonymous Volumes

**Create anonymous volume using docker run**

```bash
docker run -v path_to_folder_on_container
```

# Read-Only Bind Mounts

Timeline: [0:51:58](https://www.notion.so/0-51-58-6a03843a0e7a45259e4864b37b2124e2)

**Create read-only bind mounts using docker run**

```bash
docker run -v /path_to_folder_on_host:path_to_folder_on_container**:ro**
```

- 可以在 host machine 更改 file ，但不能在 container 裡更改。

# Loading Environment Variables From File

Timeline: [0:59:16](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=3556s)

```bash
docker run --env-file ./paht_to_env_file
```

# Deleting Stale Volumes

- 使用 anonymous volumes 建立 volumes 的話，每次 container run 都會建立新的 volumes。
- 而使用 `docker rm <container> -f` 不會刪掉跟其對應的 volumes，多加 `-v` 可連同 volumes 一起刪掉， `docker rm <container> -f**v**`

```bash
volume rm <volume_name>
or
volume prune (刪掉全部 stale volumes)
```

# Docker Compose

Timeline: [1:04:01](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=3841s)

Document: [https://docs.docker.com/compose/reference/up/](https://docs.docker.com/compose/reference/up/) and [https://docs.docker.com/compose/reference/down/](https://docs.docker.com/compose/reference/down/)

### Up

```bash
docker-compose -f <file_name> up
```

[Options](https://www.notion.so/05c2ebf081524868812bfd80788d3f19)

- up 會先在 local 尋找 docker-compose 指定的 image ，當找不到時才會去 pull or build image，因此當 image 已經 build 過，就算 file 裡指定的用來 build image 的 Dockerfile 改變，也不會重 build，可以透過 `—-build` option 去強制。
- naming convention: 用 docker compose build 的 image or up 的 container，會自動被加上 project name 前綴 (project directory name by default)
    - image name: `<project_name>_<service_name>` 。
    - container name: `<project_name>_<service_name>_<count>`。(count → 可能起了多個同 name 的 service)

### Down

```bash
docker-compose -f <file_name> down
```

**Options**

`-v`: Remove named volumes declared in the `volumes` section of the Compose file and anonymous volumes attached to containers.

`--renew-anon-volumes`: Recreate anonymous volumes instead of retrieving data from the previous containers.

- Stops containers and removes containers, networks, volumes, and images created by up.
- Networks and volumes defined as external are never removed.

# Development vs Production configs

Timeline: [1:21:36](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=4896s)

- 可以在 docker-compose 中定義 run container 的 `command`，會覆蓋掉 Dockerfile 裡的 `CMD`。

### 如何讓多個環境共用一個 base 的 docker-compose

```bash
docker-compose -f <base_compose> -f <second_compose>
```

- 後面指定的 compose 中的 key 如有跟前面指定的 compose 有衝突，則該 key 的 value 會覆寫前面的 compose 所指定的。

### Dockerfile

- Dockerfile 中如要執行 if 在 `[]` 中寫的條件前後要有空白:`Run if [ "$NODE_ENV" = "development" ]; ...`
- `ARG`: 可以指定參數。在 compose 中可以在 build 底下的 args key 傳入。

# Part 02: Working with multiple containers

# Communicating between containers

Timeline: [2:01:48](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=7308s)

1. IP Address
- docker-compose 中如未指定 network(?)，當 up 時會自動建立 default network 並將 file 裡定義的 service 都放進去同一 network，再分配 IP address 給每個 service。
    - 但當 docker-compose 重新 up 後，不保證 service 會分配到跟之前同樣的 IP address。就算有方法保證分配到同樣的 IP，但第一次時一樣要透過 docker inspect 去找出被分配的 IP address。

2. service_name/container_name

- Only exists when using custom bridge/... networks. (won't work if using default bridge/... network)
    - 建立 custom networks 會有 DNS，所以在同個 custom network 可以用 service_name/container_name 跟彼此溝通。

### 查看 container 詳細資訊

```bash
docker inspect <container_id>
```

# Container bootup order

Timeline: [2:21:45](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=8505s)

- 當 docker-compose 裡的 container 並沒有啟動順序，當有 container 需要依賴其他 container 時，可以使用 `depends_on` key 去指定。

# Nginx for Load balancing to multiple node containers

Timeline: [](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=8505s)[3:40:48](https://www.youtube.com/watch?v=9zUHg7xjIqQ&t=13248s)

### nginx conf example:

- **可指定連到 compose 中的 service_name (node-app) 來達到 load balancing**。可能有 scale 多個同樣 service，但都可用 service_name 去指定到所有同樣 service name 的 container。

```bash
service {
  listen 80;

  location / {
    proxy_pass http://node-app
    ...
  }
}
```

### Scale

```bash
docker-compose up --scale <serivce_name>=<amount>
```

# Reference
1. [Learn Docker - DevOps with Node.js & Express.](https://www.youtube.com/watch?v=9zUHg7xjIqQ)
