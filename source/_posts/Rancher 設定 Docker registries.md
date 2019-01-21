---
title: Rancher 設定 Docker registries
date: 2017-11-21 20:00:00
---

![](http://hackmd.webpat.co/uploads/upload_b21dad6d6fc52d7b20637cfdc8c59c05.png)


## 實驗環境

* **Rancher** v1.6.5
* **Docker** v17.06.1-ce

---

## 適用環境

部署 Rancher 的機器，需要增加私有 Docker registry，且該 registry 有以下特性。

* 欲加入的 server 是同網段下的 DNS
* 欲加入的 server 使用 HTTPS 做預設連線

### 錯誤訊息釋例

前述條件，如 **gitlab.ltc** 便是此一設定。同網段底下的其它機器若需要連線至 **gitlab.ltc**，便需要下載其憑證。在下圖中，使用了 **gitlab.ltc** 的 image，便會出現其錯誤訊息。

![](http://hackmd.webpat.co/uploads/upload_3964668990e662161a0be6137f971187.png)

> 錯誤訊息： Failed to allocate instance [container:xxxxxxx]: Bad instance [container:xxxxxxx] in state [error]: Error response from daemon: Get https://gitlab.ltc:4567/v2/: x509: certificate signed by unknown authority

---

## 相關解決方案

### Step 1. 加入 registries

到 Rancher 的界面中，點選 *INFRASTRUCTURE > Registries*，加入 docker registry 的 server address 及 user 的登入資訊。

![](http://hackmd.webpat.co/uploads/upload_c94a12c6d12bb38299cb9a6dc7de742e.png)


### Step 2. 下載憑證

延續前小段，接下來有兩種解決方案，皆必須進入到 Rancher 主機的 console，分別為以下兩種：

| | 套用範圍 | 生效方式 |
|---|---|---|
| 方案 A | docker | 直接生效 |
| 方案 B | 全域設定 | 重啟 docker |

其中方案一特別適合在已有運行中 containers 的機器上，因此不適合重啟 docker；而方案二適合在該機器中，還會有其它服務、程式會連線至相同 server。

#### 方案 A （不須重啟，僅 docker 生效）

##### A-1. 產生 client 端的 certificates

此階段需要產生三份檔案: **client.key**、**client.cert** 與 **ca.crt**。

```shell=
# Creating the client certificates.
$ openssl genrsa -out client.key 4096
$ openssl req -new -x509 -text -key client.key -out client.cert
$ echo -n | \
    openssl s_client -connect gitlab.ltc:4567 | \
    sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > ./ca.crt
```

##### A-2. 將檔案移至 `/etc/docker/certs.d`

```shell=
# Create folder and copy certificates into folder.
$ sudo mkdir "/etc/docker/certs.d/gitlab.ltc:4567"
# required
$ sudo mv ./ca.crt "/etc/docker/certs.d/gitlab.ltc:4567"
# optional
$ sudo mv ./client.key "/etc/docker/certs.d/gitlab.ltc:4567"
# optional
$ sudo mv ./client.cert "/etc/docker/certs.d/gitlab.ltc:4567"
```

參考文件: [Verify repository client with certificates][solution1]


#### 方案 B （須重啟，全域生效）

##### B-1. 產生 certificates

此階段需要產生一份檔案: **ca.crt**。

```shell=
# Creating the certificates.
$ echo -n | \
    openssl s_client -connect gitlab.ltc:4567 | \
    sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > ./ca.crt
```

##### B-2. 將檔案移至 `/usr/local/share/ca-certificates/`

```shell=
# Copy certificate into folder.
$ sudo mv ./ca.crt "/usr/local/share/ca-certificates/gitlab.ltc.crt"
```

##### B-3. 套用並生效

```shell=
# Update CA (include the remove).
update-ca-certificates --fresh
# Restart the docker.
service docker restart
```


參考文件: [Adding trusted root certificates to the server][solution2]

##### 方案 B 的懶人包

以下的 shell 已將方案二實做，透過以下指令能夠快速加入憑證至本機中。

於 [CI/CD 樣板][gitlab-ci-cd-template] 的 `/manager/gitlab_certificate.sh`。而這項設定是全域性的。

```shell=
# !!! 請注意，該 shell 會進行 docker restart !!!
# 不須下載，可直接執行。
$ curl -sSL https://gitlab.webpat.co/salmon/gitlab-ci-cd-template/raw/master/managers/gitlab_certificate.sh | sudo sh
# 手動下載後，執行之。
$ bash ./gitlab_certificate.sh
```

[solution1]: https://docs.docker.com/engine/security/certificates/#understanding-the-configuration "Verify repository client with certificates"
[solution2]: http://manuals.gfi.com/en/kerio/connect/content/server-configuration/ssl-certificates/adding-trusted-root-certificates-to-the-server-1605.html "Adding trusted root certificates to the server"
[gitlab-ci-cd-template]: https://gitlab.webpat.co/salmon/gitlab-ci-cd-template "GitLab CI/CD template"


#### 其它方案 (不推薦)

##### 設定 daemon.json

於 `/etc/docker/daemon.json` 新增此段：

```json
{
  "insecure-registries" : [ "gitlab.ltc:4567" ]
}
```


### Step 3. 設定完成

![](http://hackmd.webpat.co/uploads/upload_77f4846a108d6e53663b78419812f257.png)
