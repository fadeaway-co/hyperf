# Docker Swarm 叢集搭建

現階段，容器技術已經相當成熟了，就算是中小型公司，也可以基於 Gitlab、Aliyun 映象倉庫服務 和 Docker Swarm 輕鬆搭建自己的 Docker 叢集服務。

## 安裝 Docker

```
curl -sSL https://get.daocloud.io/docker | sh
# curl -sSL https://get.docker.com/ | sh
```

修改檔案 `/lib/systemd/system/docker.service`，允許使用 `TCP` 連線 `Docker`

```
ExecStart=/usr/bin/dockerd -H unix:// -H tcp://0.0.0.0:2375
```

### 配置倉庫映象地址

基於跨國線路訪問速度過慢等問題，我們可以為 Docker 配置倉庫映象地址，來改善這些網路問題，如 [阿里雲(Aliyun) Docker 映象加速器](https://help.aliyun.com/document_detail/60750.html)，我們可以申請一個 `Docker` 加速器，然後配置到伺服器上的 `/etc/docker/daemon.json` 檔案，新增以下內容，然後重啟 `Docker`，下面的地址請填寫您自己獲得的加速器地址。

```json
{"registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"]}
```

## 搭建 Gitlab 服務

### 安裝 Gitlab

#### 修改 sshd 預設埠號

首先我們需要修改一下伺服器的 `sshd` 服務的埠號，把預設的 `22` 埠改為 `2222` 埠(或其它未被佔用的埠)，這樣可以讓 `gitlab` 通過使用 `22` 埠來進行 `ssh` 連線。

```
$ vim /etc/ssh/sshd_config

# 預設 Port 改為 2222
Port 2222

# 重啟服務
$ systemctl restart sshd.service
```

重新登入機器

```
ssh -p 2222 root@host 
```

#### 安裝 Gitlab

我們來通過 Docker 啟動一個 Gitlab 服務，如下：

```
sudo docker run -d --hostname gitlab.xxx.cn \
--publish 443:443 --publish 80:80 --publish 22:22 \
--name gitlab --restart always --volume /srv/gitlab/config:/etc/gitlab \
--volume /srv/gitlab/logs:/var/log/gitlab \
--volume /srv/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:latest
```

首次登入 `Gitlab` 需要重置密碼，預設使用者名稱為 `root`。

### 安裝 gitlab-runner

> 這裡建議與 `Gitlab` 伺服器分開部署，專門提供單獨的 runner 伺服器。

我們以 `CentOS` 的的安裝方式為例，其餘可參考 [Gitlab 官網文件](https://docs.gitlab.com/runner/install/linux-repository.html)

```
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
yum install gitlab-runner
```

當然，也可以用 `curl https://setup.ius.io | sh` 命令，更新為最新的 `git` 源，然後直接使用 yum 安裝 git 和 gitlab-runner。

```
$ curl https://setup.ius.io | sh
$ yum -y install git2u
$ git version
$ yum install gitlab-runner
```

### 註冊 gitlab-runner

通過 `gitlab-runner register --clone-url http://your-ip/` 命令來將 gitlab-runner 註冊到 Gitlab 上，注意要替換 `your-ip` 為您的 Gitlab 的內網 IP，如下：

```
$ gitlab-runner register --clone-url http://your-ip/

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://gitlab.xxx.cc/
Please enter the gitlab-ci token for this runner:
xxxxx
Please enter the gitlab-ci description for this runner:
xxx
Please enter the gitlab-ci tags for this runner (comma separated):
builder
Please enter the executor: docker-ssh, shell, docker+machine, docker-ssh+machine, docker, parallels, ssh, virtualbox, kubernetes:
shell
```

### 修改 gitlab-runner 併發執行個數

```
$ vim /etc/gitlab-runner/config.toml
concurrent = 5
```

### 修改郵箱

如果需要 `Gitlab` 傳送郵件（比如使用者建立的郵件等），可以嘗試修改 `/srv/gitlab/config/gitlab.rb`

```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "git@xxxx.com"
gitlab_rails['smtp_password'] = "xxxx"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = 'git@xxxx.com'
gitlab_rails['smtp_domain'] = "exmail.qq.com"
```

## 初始化 Swarm 叢集

登入另外一臺機器，初始化叢集

```
$ docker swarm init
```

建立自定義 Overlay 網路

```
docker network create \
--driver overlay \
--subnet 12.0.0.0/8 \
--opt encrypted \
--attachable \
default-network
```

加入叢集

```
# 顯示manager節點的TOKEN
$ docker swarm join-token manager
# 加入manager節點到叢集
$ docker swarm join --token <token> ip:2377

# 顯示worker節點的TOKEN
$ docker swarm join-token worker
# 加入worker節點到叢集
$ docker swarm join --token <token> ip:2377
```

然後配置釋出用的 gitlab-runner

> 其他與 builder 一致，但是 tag 卻不能一樣。線上環境可以設定為 tags，測試環境設定為 test

## 安裝其他應用 

以下以 `Mysql` 為例，直接使用上述 `network`，支援容器內使用 name 互調。

```
docker run --name mysql -v /srv/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=xxxx -p 3306:3306 --rm --network default-network -d mysql:5.7
```

## 安裝 Portainer

[Portainer](https://github.com/portainer/portainer)

```
docker service create \
    --name portainer \
    --publish 9000:9000 \
    --replicas=1 \
    --constraint 'node.role == manager' \
    --mount type=volume,src=portainer_data,dst=/data \
    --mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
    portainer/portainer
```

## 建立一個 Demo 專案

登入 Gitlab 建立一個 Demo 專案。並匯入我們的專案 [hyperf-skeleton](https://github.com/hyperf/hyperf-skeleton)


## 配置映象倉庫

> 我們直接使用阿里雲的即可

首先建立一個名稱空間 test_namespace，然後建立一個映象倉庫 demo，並使用本地倉庫。

然後到我們直接打包用的伺服器中，登入阿里雲 Docker Registry

```
usermod -aG docker gitlab-runner
su gitlab-runner
docker login --username=your_name registry.cn-shanghai.aliyuncs.com
```

修改我們專案裡的 .gitlab-ci.yml

```
variables:
  PROJECT_NAME: demo
  REGISTRY_URL: registry.cn-shanghai.aliyuncs.com/test_namespace
```

還有 deploy.test.yml，需要仔細對比以下檔案哦。

```yml
version: '3.7'
services:
  demo:
    image: $REGISTRY_URL/$PROJECT_NAME:test
    environment:
      - "APP_PROJECT=demo"
      - "APP_ENV=test"
    ports:
      - 9501:9501
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 5
      update_config:
        parallelism: 2
        delay: 5s
        order: start-first
    networks:
      - default-network
    configs:
      - source: demo_v1.0
        target: /opt/www/.env
configs:
  demo_v1.0:
    external: true
networks:
  default-network:
    external: true
```

然後在我們的 Portainer 中，建立對應的 Config `demo_v1.0`。當然，以下引數需要根據實際情況調整，因為我們的 Demo 中，沒有任何 I/O 操作，所以填預設的即可。

```
APP_NAME=demo

DB_DRIVER=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=hyperf
DB_USERNAME=root
DB_PASSWORD=
DB_CHARSET=utf8mb4
DB_COLLATION=utf8mb4_unicode_ci
DB_PREFIX=

REDIS_HOST=localhost
REDIS_AUTH=
REDIS_PORT=6379
REDIS_DB=0
```

因為我們配置的 gitlab-ci.yml 會檢測 test 分支和 tags，所以我們把修改的內容合併到 test 分支，然後推到 gitlab 上。

接下來我們就可以訪問叢集任意一臺機器的 9501 埠。進行測試了

```
curl http://127.0.0.1:9501/
```

## 安裝 KONG 閘道器

通常情況下，Docker Swarm 叢集是不會直接對外暴露提供訪問的，所以我們可以在上層構建一個閘道器服務，這裡推薦使用 `KONG` 作為閘道器。
還有另外一個原因是 Docker Swarm 的 `Ingress 網路` 存在設計的缺陷，在連線不復用的情況下，會有併發瓶頸，具體細節請檢視對應的 `Issue` [#35082](https://github.com/moby/moby/issues/35082)
而 `KONG` 作為閘道器服務，預設情況下會複用後端的連線，所以會極大減緩上述問題。

### 安裝資料庫

```
docker run -d --name kong-database \
  --network=default-network \
  -p 5432:5432 \
  -e "POSTGRES_USER=kong" \
  -e "POSTGRES_DB=kong" \
  -e "POSTGRES_PASSWORD=kong" \
  postgres:9.6
```

### 安裝閘道器 

初始化資料庫

```
docker run --rm \
  --network=default-network \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_PASSWORD=kong" \
  -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
  kong:latest kong migrations bootstrap
```

啟動

```
docker run -d --name kong \
  --network=default-network \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_PASSWORD=kong" \
  -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  kong:latest
```

### 安裝 KONG Dashboard

> 暫時 `Docker` 中沒有更新 `v3.6.0` 所以最新版的 `KONG` 可能無法使用，可以使用 0.14.1 版本的 KONG

```
docker run --rm --network=default-network -p 8080:8080 -d --name kong-dashboard pgbi/kong-dashboard start \
  --kong-url http://kong:8001 \
  --basic-auth user1=password1 user2=password2
```

### 配置 Service

接下來只需要把部署 `KONG` 閘道器的機器 `IP` 對外暴露訪問，然後配置對應的 `Service` 即可。
如果機器直接對外暴露訪問，那麼最好只開放 `80` 和 `443` 埠，然後把 `Kong` 容器的 `8000` 和 `8443` 埠對映到 `80` 和 `443` 埠上。
當然，如果使用了 `SLB` 等負載均衡服務，也直接通過負載均衡，把 `80` 和 `443` 埠對映到 `KONG` 所在機器的 `8000` `8443` 埠上。

## 如何使用 Linux Crontab

`Hyperf` 雖然提供了 `crontab` 元件，但可能並不一定可以滿足所有人的需求，這裡提供一個 `Linux` 下使用的指令碼，來執行 `Docker` 內的 `Command`。

```bash
#!/usr/bin/env bash
basepath=$(cd `dirname $0`; pwd)
docker pull registry-vpc.cn-shanghai.aliyuncs.com/namespace/project:latest
docker run --rm -i -v $basepath/.env:/opt/www/.env \
--entrypoint php registry-vpc.cn-shanghai.aliyuncs.com/namespace/project:latest \
/opt/www/bin/hyperf.php your_command
```

## 核心優化

> 本小節內容，有待驗證，謹慎使用

安裝 `KONG` 閘道器時，有介紹 `Ingress 網路` 存在設計的缺陷，這塊可以通過 `優化核心` 處理。

- 指定 TLinux 源

```
tee /etc/yum.repos.d/CentOS-TLinux.repo <<-'EOF' 
[Tlinux]
name=Tlinux for redhat/centos $releasever - $basearch
failovermethod=priority
gpgcheck=0
gpgkey=http://mirrors.tencentyun.com/epel/RPM-GPG-KEY-EPEL-7
enabled=1
baseurl=https://mirrors.tencent.com/tlinux/2.4/tlinux/x86_64/
EOF
```

- 安裝指定核心

```
yum -y install kernel-devel-4.14.105-19.0012.tl2.x86_64 kernel-4.14.105-19.0013.tl2.x86_64 kernel-headers-4.14.105-19.0013.tl2.x86_64
```

- 使核心生效

```
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
```

- 重啟機器

```
reboot
```

### 容器引數優化

> 需要 Docker 19.09.0 以上支援，與 image 配置同級

```yaml
sysctls:
  # 網路連線複用模式的選擇
  - net.ipv4.vs.conn_reuse_mode=0
  # 當LVS轉發資料包，發現目的RS無效（刪除）時，會丟棄該資料包，但不刪除相應連線。值為1時，則馬上釋放相應連線
  - net.ipv4.vs.expire_nodest_conn=1
```

## 常見問題

### fatal: git fetch-pack: expected shallow list

這種情況是 `gitlab-runner` 使用的 `git` 版本過低，更新 `git` 版本即可，如下：

```
$ curl https://setup.ius.io | sh
$ yum remove -y git
$ yum -y install git2u
$ git version

# 重新安裝 gitlab-runner 並重新註冊 gitlab-runner
$ yum install gitlab-runner
```

### Service 重啟後，內網出現偶發的，容器無法觸達的問題，比如多次在其他容器，訪問此服務的介面，會出現 Connection refused

這是由於 IP 不夠用導致，可以修改網段，增加可用 IP

建立新的 Network

```
docker network create \
--driver overlay \
--subnet 12.0.0.0/8 \
--opt encrypted \
--attachable \
default-network
```

為服務增加新的 Network

```
docker service update --network-add default-network service_name
```

刪除原來的 Network

```
docker service update --network-rm old-network service_name
```

### 為 Service 增加節點，發現一直卡在 create 階段

原因和解決辦法同上

### 當在 Portainer 中修改了倉庫密碼後，更新 Service 失敗

這是因為 Portainer 修改後，不能作用於已經建立的服務，所以手動更新即可

```
docker service update --with-registry-auth service_name   
```
