---
title: "アウトバウンドトラフィックのIP範囲除外"
description: "サイドカーインターセプトからのアウトバウンドトラフィックの IP アドレス範囲の除外"
type: docs
weight: 5
---
このガイドでは、アウトバウンドのIPアドレス範囲をosm-edgeのプロキシサイドカーによる傍受から除外し、サービスメッシュフィルタリングとルーティングポリシーの対象としない方法を示す。

## 前提条件
- Kubernetesクラスターバージョン {{< param min_k8s_version >}}あるいはそれより高いバージョン。
- osm-edgeはインストールされている。
- API サーバーとやり取りするためのは`kubectl`使用可能。
- サービスメッシュを管理するための `osm` CLI は利用可能。

## デモ

次のデモは、IP アドレスを直接に使用して httpbin.org ウェブサイトに HTTP リクエストを作成する HTTP `curl`クライアントを示す。非メッシュの宛先 (このデモでは「httpbin.org」) へのトラフィックがポッドから送信できないように、送信機能を明示的に無効にする。

1. メッシュ全体のエグレスパススルーを無効にする。
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```

2. その名前空間をメッシュに登録した後、`curl` クライアントを `curl` 名前空間にデプロイする。

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    `curl` クライアント ポッドが稼働中であることを確認する。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

3. `httpbin.org` ウェブサイトの公共 IP アドレスを取得する。このデモでは、トラフィック傍受の対象から除外するIPレンジを1つ指定してテストする。この例では、IP 範囲「54.91.118.50/32」で表される IP アドレス「54.91.118.50」を使用して、送信 IP 範囲の除外が設定されている場合とされていない場合の HTTP リクエストを作成する。
    ```console
    $ nslookup httpbin.org
    Server:		172.23.48.1
    Address:	172.23.48.1#53

    Non-authoritative answer:
    Name:	httpbin.org
    Address: 54.91.118.50
    Name:	httpbin.org
    Address: 54.166.163.67
    Name:	httpbin.org
    Address: 34.231.30.52
    Name:	httpbin.org
    Address: 34.199.75.4
    ```

    > 注意: `54.91.118.50` を、以降の手順で上記のコマンドによって返される有効な IP アドレスに置き換える。

4. `curl` クライアントが、`http://54.91.118.50:80` で実行されている `httpbin.org`ウェブサイトに対して正常な HTTP リクエストを作成できないことを確認する。

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.91.118.50:80
    curl: (7) Failed to connect to 54.91.118.50 port 80: Connection refused
    command terminated with exit code 7
    ```

    デフォルトでは、アウトバウンドトラフィックは`curl`クライアントのポッド上で動作するPipyプロキシサイドカーを介してリダイレクトされ、プロキシはこのトラフィックを許可しないサービスメッシュポリシーに従うので、上記の失敗は予想される。

5. IP range54.91.118.50/32` の IP 範囲を除外するように osm-edge をプログラムする。
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["54.91.118.50/32"]}}}'  --type=merge
    ```

6. MeshConfigが想定通りに更新されたことを確認する。
    ```console
    # 54.91.118.50 is one of the IP addresses of httpbin.org
    $ kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
    ["54.91.118.50/32"]
    ```

7.  更新された送信IP範囲除外を設定できるように、curlクライアントPodを再起動する。トラフィックインターセプトルールは、ポッドの作成時にのみ init コンテナーによってプログラムされるため、更新された設定を取得するには、既存のポッドを再起動する必要があることに注意してください。
    ```bash
    kubectl rollout restart deployment curl -n curl
    ```

    再起動されたポッドが稼働するまで待つ。

8. `curl` クライアントが、`http://54.91.118.50:80` で実行されている `httpbin.org` ウェブサイトへの HTTP リクエストを成功させることができることを確認する。
    ```console
    # 54.91.118.50 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.91.118.50:80
    HTTP/1.1 200 OK
    Date: Thu, 18 Mar 2021 23:17:44 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 9593
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    ```

9. 除外されていない「httpbin.org」ウェブサイトの他の IP アドレスへの HTTP リクエストが失敗することを確認する。
    ```console
    # 34.199.75.4 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://34.199.75.4:80
    curl: (7) Failed to connect to 34.199.75.4 port 80: Connection refused
    command terminated with exit code 7
    ```