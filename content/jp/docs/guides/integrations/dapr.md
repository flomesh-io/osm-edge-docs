---
Title: 「Dapr と osm-edge の統合」
Description: "Dapr と osm-edge の統合を示す簡単なデモ"
Alias: "/docs/integrations/demo_dapr"
Type: docs
Weight: 2
---

# Dapr を osm-edge と統合する

## Dapr osm-edge ウォークスルー

このドキュメントでは、Dapr を Kubernetes クラスターの osm-edge と連携させる手順について説明します。

1. mTLS を無効にしてクラスターに Dapr をインストールします。

   1. Dapr には、ユーザーが dapr とその機能に慣れるのに役立つクイックスタート リポジトリがあります。 この統合デモでは、[hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes) クイックスタートを利用します。 この Dapr の例を osm-edge と統合したいので、いくつかの変更が必要です。それらは次のとおりです。

      - [hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes) デモでは、(デフォルトで) mtls を有効にして Dapr をインストールします。 これにはosm-edgeを活用したい**。 したがって、クラスターに [installing Dapr](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-1---setup-dapr-on-your-kubernetes-cluster) しながら、 インストール中にフラグ「--enable-mtls=false」を渡して mtls を無効にしてください。
       - さらに [hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes) は、デフォルトの名前空間にすべてを設定します。hello 全体を設定することを **強くお勧めします** -特定の名前空間での kubernetes デモ (後でこの名前空間を osm-edge のメッシュに結合します)。 この統合のために、名前空間を「dapr-test」としています。

        ```console
         $ kubectl create namespace dapr-test
         namespace/dapr-test created
        ```

      - [redis state store](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-2---create-and-configure-a-state-store)、[redis. yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/redis.yaml)、[node.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/node.yaml) と[python.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/python.yaml) をデプロイする必要があります 「dapr-test」名前空間
       - このデモのリソースはカスタム名前空間に設定されているため。 Dapr がシークレットにアクセスできるようにするには、クラスタに rbac ルールを追加する必要があります。 次のロールとロール バインディングを作成します。

        ```bash
        kubectl apply -f - <<EOF
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: secret-reader
          namespace: dapr-test
        rules:
        - apiGroups: [""]
          resources: ["secrets"]
          verbs: ["get", "list"]
        ---

        kind: RoleBinding
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
          name: dapr-secret-reader
          namespace: dapr-test
        subjects:
        - kind: ServiceAccount
          name: default
        roleRef:
          kind: Role
          name: secret-reader
          apiGroup: rbac.authorization.k8s.io
        EOF
        ```

   2. サンプル アプリケーションが必要に応じて Dapr で実行されていることを確認します。

2. osm-edge をインストールします。

   ```console
   $ osm install
   osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
   ```

3. osm-edge で許可モードを有効にします。

   ```console
   $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
   meshconfig.config.openservicemesh.io/osm-mesh-config patched
   ```

   これは、hello-kubernetes の例がそのまま機能し、最初から SMI ポリシーが不要になるようにするために必要です。

4. kubernetes API サーバー IP が osm-edge のサイドカーによってインターセプトされないように除外します。

   1. kubernetes API サーバー クラスター IP を取得します。
      ```console
      $ kubectl get svc -n default
      NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
      kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   1d
      ```
   2. この IP を MeshConfig に追加して、この IP へのアウトバウンド トラフィックが osm-edge のサイドカーによる傍受から除外されるようにします
      ```console
      $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["10.0.0.1/32"]}}}'  --type=merge
      meshconfig.config.openservicemesh.io/osm-mesh-config patched
      ```

   Dapr は Kubernetes シークレットを利用してこのデモの redis 状態ストアにアクセスするため、osm-edge で Kubernetes API サーバー IP を除外する必要があります。

   _注: Dapr コンポーネント ファイルにパスワードをハードコーディングした場合は、この手順を省略できます。_

5. ポートが osm-edge のサイドカーによってインターセプトされないようにグローバルに除外します。

   1. Dapr の配置サーバー (`dapr-placement-server`) のポートを取得します。
      ```console
      $ kubectl get svc -n dapr-system
      NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
      dapr-api                ClusterIP   10.0.172.245   <none>        80/TCP               2h
      dapr-dashboard          ClusterIP   10.0.80.141    <none>        8080/TCP             2h
      dapr-placement-server   ClusterIP   None           <none>        50005/TCP,8201/TCP   2h
      dapr-sentry             ClusterIP   10.0.87.36     <none>        80/TCP               2h
      dapr-sidecar-injector   ClusterIP   10.0.77.47     <none>        443/TCP              2h
      ```
   2. [redis.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/redis.yaml) から redis ステート ストアのポートを取得します (このデモの場合は「6379」)。

   3. これらのポートを MeshConfig に追加して、そこへのアウトバウンド トラフィックが osm-edge のサイドカーによるインターセプトから除外されるようにします

      ```console
      $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundPortExclusionList":[50005,8201,6379]}}}'  --type=merge
      meshconfig.config.openservicemesh.io/osm-mesh-config patched
      ```

   Dapr の配置サーバー (`dapr-placement-server`) ポートが osm-edge のサイドカーによってインターセプトされないようにグローバルに除外する必要があります。これは、Dapr を持つポッドが Dapr のコントロール プレーンと通信する必要があるためです。 Dapr のサイドカーが osm-edge のサイドカーによってインターセプトされることなく、トラフィックを redis にルーティングできるように、redis ステート ストアも除外する必要があります。

   _注: ポートをグローバルに除外すると、osm-edge のメッシュ内のすべてのポッドが、指定されたポートへのアウトバウンド トラフィックを妨害しなくなります。 Dapr を実行している Pod でのみポートを選択的に除外する場合は、この手順を省略して、以下の手順に従ってください。_

6. Pod レベルでポートが osm-edge のサイドカーによってインターセプトされないように除外します。

   1. Dapr の API とセントリー (`dapr-sentry` と `dapr-api`) のポートを取得します。

      ```console
      $ kubectl get svc -n dapr-system
      NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
      dapr-api                ClusterIP   10.0.172.245   <none>        80/TCP               2h
      dapr-dashboard          ClusterIP   10.0.80.141    <none>        8080/TCP             2h
      dapr-placement-server   ClusterIP   None           <none>        50005/TCP,8201/TCP   2h
      dapr-sentry             ClusterIP   10.0.87.36     <none>        80/TCP               2h
      dapr-sidecar-injector   ClusterIP   10.0.77.47     <none>        443/TCP              2h
      ```

   2. nodeapp ([node.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/node.yaml)) と pythonapp ([python.yaml]( https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/python.yaml)) に次の注釈を含めます: `openservicemesh.io/outbound-port-exclusion-list: "80"`

   Pod にアノテーションを追加すると、Dapr の api (`dapr-api`) および sentry (`dapr-sentry`) ポートが osm-edge のサイドカーによってインターセプトされるのを防ぎます。これらの Pod は Dapr のコントロール プレーンと通信する必要があるためです。

7. Dapr hello-kubernetes デモのセットアップに使用された名前空間を osm-edge に監視させます。

   ```console
   $ osm namespace add dapr-test
   Namespace [dapr-test] successfully added to mesh [osm]
   ```

8. Dapr hello-kubernetes ポッドを削除して再デプロイします。

   ```console
   $ kubectl delete -f ./deploy/node.yaml
   service "nodeapp" deleted
   deployment.apps "nodeapp" deleted
   ```

   ```console
   $ kubectl delete -f ./deploy/python.yaml
   deployment.apps "pythonapp" deleted
   ```

   ```console
   $ kubectl apply -f ./deploy/node.yaml
   service "nodeapp" created
   deployment.apps "nodeapp" created
   ```

   ```console
   $ kubectl apply -f ./deploy/python.yaml
   deployment.apps "pythonapp" created
   ```

   再起動時の pythonapp と nodeapp ポッドにはそれぞれ 3 つのコンテナーがあり、osm-edge のプロキシ サイドカーが正常に挿入されたことを示します。

   ```console
   $ kubectl get pods -n dapr-test
   NAME                         READY   STATUS    RESTARTS   AGE
   my-release-redis-master-0    1/1     Running   0          2h
   my-release-redis-slave-0     1/1     Running   0          2h
   my-release-redis-slave-1     1/1     Running   0          2h
   nodeapp-7ff6cfb879-9dl2l     3/3     Running   0          68s
   pythonapp-6bd9897fb7-wdmb5   3/3     Running   0          53s
   ```

9. Dapr hello-kubernetes デモが期待どおりに動作することを確認します。

   1. [こちら](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-3---deploy-the-nodejs-app-with-the-dapr-sidecar) に記載されている手順を使用して、nodeapp サービスを確認します。

   2. [こちら](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-6---observe-messages) に記載されている pythonapp を確認します。

10. SMI トラフィック ポリシーの適用:

    これまでのデモでは、メッシュ内のアプリケーション接続が「osm-controller」によって自動的に設定される osm-edge の許容トラフィック ポリシー モードを示していたため、pythonapp が nodeapp と通信するために SMI ポリシーは必要ありませんでした。

     SMI トラフィック ポリシーを使用した同じデモの動作を確認するには、次の手順に従ってください。

    1. 許可モードを無効にします。

       ```console
       $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
       meshconfig.config.openservicemesh.io/osm-mesh-config patched
       ```

    2. [こちら](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-6---observe-messages)に記載されている pythonapp によってオーダー ID がインクリメントされなくなったことを確認します。

    3. nodeapp と pythonapp のサービス アカウントを作成します。

       ```console
       $ kubectl create sa nodeapp -n dapr-test
       serviceaccount/nodeapp created
       ```

       ```console
       $ kubectl create sa pythonapp -n dapr-test
       serviceaccount/pythonapp created
       ```

    4. クラスタのロール バインディングを更新して、新しく作成されたサービス アカウントを含めます。

       ```bash
       kubectl apply -f - <<EOF
       ---
       kind: RoleBinding
       apiVersion: rbac.authorization.k8s.io/v1
       metadata:
         name: dapr-secret-reader
         namespace: dapr-test
       subjects:
       - kind: ServiceAccount
         name: default
       - kind: ServiceAccount
         name: nopdeapp
       - kind: ServiceAccount
         name: pythonapp
       roleRef:
         kind: Role
         name: secret-reader
         apiGroup: rbac.authorization.k8s.io
       EOF
       ```

    5. 次の SMI アクセス コントロール ポリシーを適用します。

        SMI TrafficTarget を展開する

       ```bash
       kubectl apply -f - <<EOF
       ---
       kind: TrafficTarget
       apiVersion: access.smi-spec.io/v1alpha3
       metadata:
         name: pythodapp-traffic-target
         namespace: dapr-test
       spec:
         destination:
           kind: ServiceAccount
           name: nodeapp
           namespace: dapr-test
         rules:
         - kind: HTTPRouteGroup
           name: nodeapp-service-routes
           matches:
           - new-order
         sources:
         - kind: ServiceAccount
           name: pythonapp
           namespace: dapr-test
       EOF
       ```

       HTTPRouteGroup ポリシーをデプロイする

       ```bash
       kubectl apply -f - <<EOF
       ---
       apiVersion: specs.smi-spec.io/v1alpha4
       kind: HTTPRouteGroup
       metadata:
         name: nodeapp-service-routes
         namespace: dapr-test
       spec:
         matches:
         - name: new-order
       EOF
       ```

    6.nodeapp ([node.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/node.yaml)) と pythonapp ([python.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/python.yaml)) の両方で Pod 仕様を更新して、それぞれのサービス アカウントを含めます。 Dapr hello-kubernetes ポッドを削除して再デプロイする 

    7. Dapr の hello-kubernetes デモが期待どおりに動作することを確認します ([こちら](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-6---observe-messages) を参照)。
   
11. 掃除: 

    1. Dapr hello-kubernetes デモをクリーンアップするには、「dapr-test」名前空間をクリーンアップします

       ```console
       $ kubectl delete ns dapr-test
       ```

    2. Dapr をアンインストールするには、次を実行します。

       ```console
       $ dapr uninstall --kubernetes
       ```

    3. osm-edge をアンインストールするには、次を実行します

       ```console
       $ osm uninstall mesh
       ```

    4. アンインストール後に osm-edge のクラスター全体のリソースを削除するには、次のコマンドを実行します。 詳細なコンテキストと情報については、[uninstall guide](docs/guides/uninstall/) を参照してください。

       ```console
       $ osm uninstall mesh --delete-cluster-wide-resources
       ```
