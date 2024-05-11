---
title: "Dockerのコンテナ内からlocalhostという名前でホストにアクセスする"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker"]
published: false
---

これは2020年ごろに書きかけだった記事を再投稿したものです。2024年現在での正確性は保証できません。

Dockerのコンテナとホストのプロセスの両方から同じ名前でホストで起動したサーバーにアクセスしたかったので調べてみました。

以下の記事を参考にさせていただきましたが、ホストのIPアドレスを意識せずに使いたかったので、別の方法も調べた上で整理しました。

参考: Dockerコンテナ内からホストへ`localhost` でアクセスしてみる - Qiita
https://qiita.com/kai_kou/items/5182965ea75c85cf1e3f


## Linuxの場合

Linuxの場合、 `docker run` に `--net host` をつけてコンテナを起動すれば、ホストと同じネットワークスタックで起動するので、 `localhost` でホストにアクセスできるようになります。Docker Composeの場合は、 `network_mode: host` を使います。

Docker for Mac/Windowsではできません。

## localhostという名前でなくても良い場合

必ずしも `localhost` というホスト名でなくても良い場合、Docker for Mac/Windowsでは `host.docker.internal` というホスト名を使えばOKです。コンテナ内からこのホストにアクセスすると、ホストにアクセスできます。

一般的な使い方であればこれで十分なケースが多いでしょう。

## ホストのIPアドレスを指定できる場合

ホストのIPアドレスは環境によって異なりますが、それを気にしなくて良い場合やシェルスクリプトなどで取得できる場合は、 `--add-host localhost:<ホストのIPアドレス>` とすればOKです。コンテナの `/etc/hosts` にこのエントリーが追加されます。

参考:
https://docs.docker.com/engine/reference/commandline/run/#add-entries-to-container-hosts-file---add-host

IPアドレスを取得してから `docker run` でコンテナを起動するようなシェルスクリプトを書いても良いですが、Docker Composeで起動する場合は不自然になるのでさらに調べてみました。

## 転送するポート数が限られている場合

ソケットをリレーしてくれるコマンド `socat` を使うのが楽そうです。この場合、複数コンテナを起動するのが自然なので、Docker Composeの利用を前提として解説します。 `network_mode: "service:<接続元コンテナのサービス>"` で接続元コンテナと同じネットワークスタックを使えるので、接続元のコンテナからはlocalhostでsocatのコンテナに接続できます。

alpine/socat - Docker Hub
https://hub.docker.com/r/alpine/socat/

```yml
version: "3.7"
services:
    socat:
        image: alpine/socat
        command: ["-d", "tcp-listen:8000,fork,reuseaddr", "tcp-connect:host.docker.internal:8000"]
        network_mode: "service:gcloud-tasks-emulator"

    gcloud-tasks-emulator:
        build: .
        ports:
            - "9090:9090"
        command: ["start", "--port=9090", "--default-queue=projects/localhost/locations/localhost/queues/default", "--max-retries=1"]
```

参考: portforwarding - How can I forward a port from one docker container to another? - Stack Overflow
https://stackoverflow.com/questions/46099874/how-can-i-forward-a-port-from-one-docker-container-to-another


## 転送するポート数が多い場合

docker-hostというトラフィックをホストに送るコンテナが使えます。iptablesで実装されており、 `network_mode: "service:<接続元コンテナのサービス>"` だとうまく動かないので、`extra_hosts` で `localhost` に別IPアドレスを指定するのが妥当でしょうか。もっとシンプルな方法がありそうな気がしますが、思いつきませんでした。 `cap_add: [ "NET_ADMIN", "NET_RAW" ]` が必要です。

qoomon/docker-host: docker container to forward all traffic to the docker host
https://github.com/qoomon/docker-host

```
version: "3.7"
services:
    gcloud-tasks-emulator:
        build: .
        ports:
            - "9090:9090"
        extra_hosts:
            - "localhost:172.16.238.2"
        networks:
            app_net:
                ipv4_address: 172.16.238.3

    docker-host:
        image: qoomon/docker-host
        cap_add: [ "NET_ADMIN", "NET_RAW" ]
        restart: on-failure
        networks:
            app_net:
                ipv4_address: 172.16.238.2
networks:
    app_net:
        driver: bridge
        ipam:
            driver: default
            config:
                - subnet: 172.16.238.0/24
```

参考: How to access host port from docker container - Stack Overflow
https://stackoverflow.com/questions/31324981/how-to-access-host-port-from-docker-container

### 補足

[entrypoint.sh](https://github.com/qoomon/docker-host/blob/master/entrypoint.sh#L41-L53)を見ればわかるように、起動時にiptablesを設定して何もしていないだけです。このため、同等の処理をコンテナのentrypointに実装すれば、単一コンテナで実現することも可能です。ただし、設定内容はちょっと変わります。

```
iptables -t nat -I POSTROUTING -j MASQUERADE
iptables -t nat -I OUTPUT -p tcp -o lo --dport "$forwarding_port" -j DNAT --to-destination "$docker_host_ipv4"
iptables -t nat -I OUTPUT -p udp -o lo --dport "$forwarding_port" -j DNAT --to-destination "$docker_host_ipv4"
```

また、 `sysctl net.ipv4.conf.all.route_localnet=1` にする必要があります。

```
        sysctls:
            net.ipv4.conf.all.route_localnet: 1
```