---
title: 註冊 GitLab Runner
date: 2018-05-08 20:00:00
---

若機器尚未安裝 GitLab 請參考 [GitLab 官方安裝說明][gitlab_installation]。

GitLab Runner 是基於 GitLab 的子服務，若尚未安裝 GitLab Runner 請參考 [GitLab Runner 安裝說明][install_runner]。倘若不確定是否已安裝 GitLab Runner，可於終端機輸入以下指令來檢查。

```shell
# 查看 GitLab Runner 版本。
$ gitlab-runner -v
```

且接下來的步驟將會利用 Docker 進行 GitLab 的 CI 流程，若尚未安裝 Docker 請參考 [Docker 安裝說明][install_docker]。倘若不確定是否已安裝 Docker，可於終端機輸入以下指令來檢查。

```shell
# 查看 Docker 版本。
$ docker version
```

<br />

## 一、Docker container runner

採用 docker container 的方式執行 GitLab runner。

### 1. 建立 container

使用官方影像 [gitlab/gitlab-runner] 建立 container。

```shell=
$ docker run -d --name gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
    --restart unless-stopped \
    gitlab/gitlab-runner:latest
```

* HOST 路徑 - `/var/run/docker.sock`
    * 為 Host 上的 docker socket
* HOST 路徑 - `/srv/gitlab-runner/config`
    * Host 指定一個空資料夾即可
    * runner 設定檔名稱為 config.toml
    * ps. 倘若該主機欲配置複數 runners，請務必調整路徑以避免重疊覆寫。


### 2. 進入 runner 列表

已註冊的 Runner 列表可於 GitLab 界面中查閱、編輯概略資訊。

> Runner 列表： 
> 　　http:\/\/\<GitLab 的 domain\>/admin/runners

進入 Runners 界面後，找到下圖之段落，高亮兩處於稍後註冊 runner 會需要鍵入的兩項資訊 **GitLab URL** 與 **registration token**。

![](http://hackmd.webpat.co/uploads/upload_05ed19f569d1335ce4629c43cdd9925c.png)

### 3. 註冊 runner

進入方才建立的 container。

```shell=
$ docker exec -it gitlab-runner /bin/bash
```

進入後執行 **gitlab-runner register** 程式，並提供 runner 的相關資訊，description 欄位可依需求修改。

```shell=
# 於 container, gitlab/gitlab-runner
$ gitlab-runner register -n \
    --url <你的 GitLab URL> \
    --registration-token <你的 registration token> \
    --executor docker \
    --docker-image docker:latest \
    --description "Docker Runner"
```

| Option | Description |
| --------------:| ----------- |
| **url** | GitLab 的 URL |
| **registration-token** | 進入管理者 runner 頁面可見，是註冊 runner 必須附上的驗證 token |
| **executor** | runner 類型 |
| **docker-image** | 預設的 image，未來每次進行 CICD pipeline 時，系統都會自動 pull 最新 image |
| **description** | 會顯示於 runner 列表，相當於 friendly name |

### 4. 註冊完成

註冊完成後，該 container 會與 GitLab 持續保持連線，若需要檢查相關資訊，能夠查看該 container 的 logs 資訊。

設定檔依前述 volume 路徑，預設檔案路徑為：`/srv/gitlab-runner/config/config.toml`。倘若該主機欲配置複數 runners，請務必調整路徑以避免重疊覆寫。

設定完成的 runner 能夠在 GitLab runners 列表中所見。


### 一次做完步驟 1~3 

```shell=
$ docker run --rm -it -v /path/to/config:/etc/gitlab-runner --name gitlab-runner gitlab/gitlab-runner register \
  --non-interactive \
  --url <你的 GitLab URL> \
  --registration-token <你的 registration token> \
  --executor "docker" \
  --docker-image docker:latest \
  --description "Docker Runner" \
```

<br />


## 二、Shell runner

直接使用指定 Host 執行 GitLab runner 的方式。

### 1. 進入 runner 列表

同前章節之 [進入 runner 列表](#2-進入-runner-列表)。

### 2. 註冊 runner


在終端機輸入以下指令，並將前一步驟的 **GitLab URL** 與 **registration token** 替換之，而 description 欄位可依需求修改。

```shell
# gitlab-runner 是安裝 GitLab 同時會附帶的程式
$ sudo gitlab-runner register -n \
    --url <你的 GitLab URL> \
    --registration-token <你的 registration token> \
    --executor shell \
    --description "Shell Runner"
```

### 3. 將 gitlab-runner 加入至 docker 群組

```shell
$ sudo usermod -aG docker gitlab-runner
```

* **\-a (\-\-append)**：添增指定用戶至指定群組，必須與 `-G` 一併使用。
* **\-G (\-\-groups)**：一新群組。

### 4. 檢查 gitlab-runner 是否有權限使用 docker

```shell
$ sudo -H -u gitlab-runner docker info
```

* **\-H (\-\-set-home)**：將環境變數 `$HOME` 切換到當前用戶。
* **\-u (\-\-user)**：指定成為某一用戶，並執行後續指令

見下圖，若能正常顯示 `docker info` 的輸出資訊，便可接續下一步驟。

![](http://hackmd.webpat.co/uploads/upload_bdb4e5a91dac96f2e4e023a2e96c863a.png)

### 5. 註冊完成

設定完成的 runner 能夠在 GitLab runners 列表中所見。


<br />

### 於 GitLab CI 使用 docker

完成前述設定（[docker container runner](#一、docker-container-runner) 或 [shell runner](#二、shell-runner)）後，便可於 `.gitlab-ci.yml` 中進行 docker 指令的調用，便於 push 該 commit 後，GitLab CI 的 runner 將能夠取用 docker 的資源。

![](http://hackmd.webpat.co/uploads/upload_9a8c47ad7dbda608e773cb095d9df146.png)

以下為上圖之 `.gitlab-ci.yml` 設定：

```yml
image: node:8.4.0

stages:
  - deploy

deploy_job:
  image: docker:latest
  stage: deploy
  script:
    - echo "Deploy okay."
```

## Q&A
### 1. 錯誤訊息 - x509: certificate signed by unknown authority

```
Error response from daemon: Missing key ca.key for client certificate ca.cert. 
Note that CA certificates should use the extension .crt.

ERROR: Preparation failed: Error response from daemon: Get https://gitlab.ltc:4567/v1/_ping: x509: certificate signed by unknown authority
```

#### 解決方案
* 下載 gitlab.ltc 憑證, 放到裝有 gitlab-runner container 的 host `/etc/docker/certs.d/gitlab.ltc:4567/ca.crt`


### 2. 錯誤訊息 - failed to get default registry endpoint from daemon

```
$ docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_SERVER
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Warning: failed to get default registry endpoint from daemon (Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?). Using system default: https://index.docker.io/v1/
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
ERROR: Job failed: exit code 1
```

#### 解決方案
* 修改 gitlab-runner config `config.toml`, 在 `[runners.docker]` 區段的 volumes 補上 `/var/run/docker.sock:/var/run/docker.sock`

* `config.toml` 設定如下

```YAML=
concurrent = 4
check_interval = 0

[[runners]]
  name = "Docker Runner"
  url = "https://gitlab.webpat.co/"
  token = "d05b450b6b4446a75b6231284e4a7a"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "docker:latest"
    privileged = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
  [runners.cache]
```


### 3. Pipeline jobs 無法共享 cache

當 runner 的 concurrent 配置大於 1 時，會發生此情形，導致後續 job 無法正確 pull 先前的 cache。

而發生錯誤的前提是：在一 pipeline 中出現前後 job 被指派給 runner 不同的 concurrent。

```
# 出現在 pipeline - job installation
Running on runner-d05b450b-project-107-concurrent-1 via 860ed6055b41...

# 出現在 pipeline - job test
Running on runner-d05b450b-project-107-concurrent-0 via 860ed6055b41...
```

#### 解決方案

透過 artifacts 達到相同的效果，首先修改相關 repo 的 ` .gitlab-ci.yml`

```yaml=
# 將 installation 階段的產物作為 artifacts
installation:
  stage: build
  artifacts: # 後續 jobs 將會繼承該 path 的檔案
    paths:
      - node_modules/
    expire_in: 1 day
  script:
    # ....
```

### 4. 錯誤訊息 - Uploading artifacts to coordinator... too large archive

```shell
# 完整訊息
ERROR: Uploading artifacts to coordinator... too large archive    id=1 token=****
FATAL: Too large
```

#### 解決方案
* https://gitlab.webpat.co/help/user/admin_area/settings/continuous_integration#maximum-artifacts-size

因 artifacts 由 runner 產出，再透過 HTTP 傳遞給 GitLab 進行儲存及管理。便需要更改 **(a)** artifacts 本身的檔案上限 與 **(b)** web server (nginx) 能夠接受的檔案大小。

```shell
# 路徑：/etc/gitlab/gitlab.rb
nginx['client_max_body_size'] = '1024m'

# 路徑：/var/opt/gitlab/nginx/conf/gitlab-http.conf
client_max_body_size 1024m;
```

最後，重新啟動 GitLab 以套用新設定。

#### 其它解決方案

若上述解決方案無法達成目的，有可能是 GitLab 服務本身在受理服務前已被 Proxy 所代理。

在不更動 Proxy 的前提下，應當使 runner 與 GitLab 之間的溝通採內網形式。相關的設定能夠在 `config.toml` 中找到。

```YAML=
[[runners]]
  name = "Docker Runner"
  # 外部網址 https://gitlab.webpat.co/
  url = "https://gitlab.ltc/"
```

> 附註，能夠透過 `docker logs gitlab-runner` 來觀察目前的配置是否正確，若無訊息輸出，表正常設定。


## 參考資料

[GitLab CI/CD 設定手冊]（HackMD）
[Registering Runners]
[Install GitLab Runner using the official GitLab repositories][install_runner]
[Install Docker][install_docker]
[GitLab Installation][gitlab_installation]

## 相關實驗

[[LAB] GitLab Runner @ Rancher](http://hackmd.webpat.co/s/BybwBrg0f#)





[gitlab/gitlab-runner]: https://hub.docker.com/r/gitlab/gitlab-runner/
[GitLab CI/CD 設定手冊]: http://hackmd.webpat.co/s/BJE9U1Y9W
[Registering Runners]: https://docs.gitlab.com/runner/register/index.html
[install_runner]: https://docs.gitlab.com/runner/install/linux-repository.html "Install GitLab Runner using the official GitLab repositories"
[install_docker]: https://docs.docker.com/engine/installation/ "Install Docker"
[gitlab_installation]: https://about.gitlab.com/installation/ "GitLab Installation"
