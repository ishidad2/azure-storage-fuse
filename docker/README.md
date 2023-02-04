# blobfuse2のビルド

```
cd ..
go get
go build -tags=fuse3
./build.sh
```

# build container

※コンテナイメージの作成

```
./buildcontainer.sh
```

# docker-composeで起動

例）

docker-compose.yml

```
version: "3"
services:
  blobfuse:
    image: ishidad2/azure-blobfuse2:2.0.2
    volumes:
      - ./myconfig.yaml:/usr/share/blobfuse2/config.yaml
    cap_add:
      - SYS_ADMIN
    devices:
      - /dev/fuse
    security_opt:
      - apparmor:unconfined
```

例）
myconfig.yaml
```
# Refer ./setup/baseConfig.yaml for full set of config parameters
allow-other: true

logging:
  type: syslog

components:
  - libfuse
  - file_cache
  - attr_cache
  - azstorage

libfuse:
  attribute-expiration-sec: 120
  entry-expiration-sec: 120
  negative-entry-expiration-sec: 240

file_cache:
  path: /tmp/blobfuse_temp
  timeout-sec: 0 
  allow-non-empty-temp: true
  cleanup-on-start: true

attr_cache:
  timeout-sec: 7200

azstorage:
#Required
  type: block
  account-name: blobfuse*** #ストレージ アカウント名
  container: fuse**** #ストレージ アカウントのコンテナー名
  endpoint: https://******.blob.core.windows.net/ #ストレージ アカウントの blob エンドポイント URL
  mode: key #利用する認証の種類
  account-key: P************************************************== #ストレージ アカウントのアクセスキー

#Disk cache related configuration
file_cache:
#Required
  path: /mnt/blobfuse_temp #キャッシュフォルダのパスを指定
```