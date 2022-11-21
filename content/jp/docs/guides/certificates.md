---
title: 「証明書管理」
description: 「osm-edge は、ポッド間のデータの暗号化、および Pipy とサービス ID に mTLS を使用します。」
type: docs
weight: 10
---

# 証明書管理

## mTLS と証明書の発行

osm-edge は、ポッド間のデータの暗号化、および Pipy とサービス ID に mTLS を使用します。 証明書が作成され、osm-edge コントロール プレーンによって SDS プロトコルを介して各 Pipy プロキシに配布されます。

## 証明書の種類

osm-edge で使用される証明書にはいくつかの種類があります。

| 証明書の種類 | 使用方法                                                                                | 有効期間                                                                              | CommonName の例 |
| ---------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| サービス          | Pipy 間の東西通信に使用されます。 サービス アカウントを識別します                  | デフォルトの「24h」; `osm.certificateProvider.serviceCertValidityDuration` インストール オプションで定義 | `bookstore-v2.bookstore.cluster.local`                         |
| Webhook サーバー   | 変更、検証、および CRD 変換 Webhook サーバーによって使用されます                           | 10年                                                                                       | `osm-injector.osm-system.svc`                                 |
|                  |


### ルート証明書

サービス メッシュのルート証明書は、osm がインストールされている名前空間 (デフォルトでは「osm-system」) の「osm-ca-bundle」という名前の不透明な Kubernetes シークレットに格納されます。
シークレット YAML の形状は次のとおりです。

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
      name: osm-ca-bundle
      namespace: osm-system
data:
  ca.crt: <base64 encoded root cert>
  expiration: <base64 encoded ISO 8601 certificate expiration date; for example: 2031-01-11T23:15:09.299Z>
  private.key: <base64 encoded private key>
```

これが使用される詳細とコードについては、[osm-controller.go](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/cmd/osm-controller/osm-controller.go#L182-L183)を参照してください。

ルート証明書を読み取るには (Hashicorp Vault を除く)、対応するシークレットを取得してデコードできます。

```console
kubectl get secret -n $osm_namespace $osm_ca_bundle -o jsonpath='{.data.ca\.crt}' |
    base64 -d |
    openssl x509 -text -noout
```

*注: デフォルトでは、CA バンドルの名前は「osm-ca-bundle」です。*

これにより、有効期限や発行者などの貴重な証明書情報が提供されます。

### ルート証明書のローテーション (Tresor)

osm-edge 内の Tresor パッケージを介して作成される自己署名ルート証明書は、10 年後に期限切れになります。 ルート証明書をローテーションするには、次の手順に従う必要があります。

1. osm 名前空間の「osm-ca-bundle」証明書を削除します
   ```console
   export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
   kubectl delete secret osm-ca-bundle -n $osm_namespace
   ```

2. コントロール プレーン コンポーネントを再起動する
   ```console
   kubectl rollout restart deploy osm-controller -n $osm_namespace
   kubectl rollout restart deploy osm-injector -n $osm_namespace
   kubectl rollout restart deploy osm-bootstrap -n $osm_namespace
   ```

コンポーネントが再デプロイされると、最終的に `$osm_namespace` で新しい `osm-ca-bundle` シークレットを確認できるはずです:

```console
kubectl get secrets -n $osm_namespace
```

```
NAME                           TYPE                                  DATA   AGE
osm-ca-bundle                  Opaque                                3      74m
```

新しい有効期限は、次のコマンドで確認できます。
```console
kubectl get secrets -n $osm_namespace osm-ca-bundle -o json | jq -r '.data.expiration' | base64 -d
```

#### その他の証明書発行者

Tresor 以外の証明書プロバイダーの場合、ルート証明書をローテーションするプロセスは異なります。 Hashicorp Vault と cert-manager.io の場合、ユーザーはルート証明書を自分で osm-edge の外でローテーションする必要があります。

## 証明書の発行

Open Service Mesh は、証明書を発行する 3 つの方法をサポートしています。

- [Tresor](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/pkg/certificate/providers/tresor) と呼ばれる内部の osm-edge パッケージを使用します。 これは、初めてインストールする場合のデフォルトです。
- [Hashicorp Vault](https://www.vaultproject.io/) を使用
- [cert-manager](https://cert-manager.io) を使用

### osm-edge の Tresor 証明書発行者を使用する

osm-edge には [tresor](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/pkg/certificate/providers/tresor) パッケージが含まれています。 これは `certificate.Manager` インターフェイスの最小限の実装です。 「crypto」Go ライブラリを利用して証明書を発行し、これらの証明書を Kubernetes シークレットとして保存します。

- 開発中に `tresor` パッケージを使用するには、このリポジトリの `.env` ファイルで `export CERT_MANAGER=tresor` を設定します。

- このパッケージを Kubernetes クラスターで使用するには、展開前に Helm チャートで「CERT_MANAGER=tresor」変数を設定します。

さらに: 

- `osm.caBundleSecretName` - この文字列は、CA ルート証明書と秘密鍵が保存される Kubernetes シークレットの名前です。

### Hashicorp Vault の使用

サービス メッシュの CA ルート キーを Kubernetes に保存するのは安全でないと考えているサービス メッシュ オペレーターには、[Hashicorp Vault](https://www.vaultproject.io/) インストールと統合するオプションがあります。 このようなシナリオでは、事前構成された Hashi Vault が必要です。 Open Service Mesh のコントロール プレーンは、Vault の URL に接続し、認証して、証明書の要求を開始します。 この設定により、Vault を正しく安全に構成する責任がオペレーターに移ります。

osm-edge を既存の Vault インストールと統合するには、次の構成パラメーターが必要です。

- ボールト アドレス
- ボールトトークン
- 証明書の有効期間

`osm install` set フラグは、osm-edge が Vault と統合する方法を制御します。 Vault で証明書を発行するには、次の「osm install」セット オプションを構成する必要があります。

- `--set osm.certificateProvider.kind=vault` - これを `vault` に設定します
- `--set osm.vault.host` - Vault サーバーのホスト名 (例: `vault.contoso.com`)
- `--set osm.vault.protocol` - Vault 接続のプロトコル (`http` または `https`)
- `--set osm.vault.token` - osm-edge が Vault に接続するために使用するトークン (これは特定の役割のために Vault サーバーで発行されます)
- `--set osm.vault.role` - Vault サーバーで作成され、Open Service Mesh 専用のロール (例: `openservicemesh`)
- `--set osm.certificateProvider.serviceCertValidityDuration` - サービス間通信のために発行された各新しい証明書が有効になる期間。 これは、オプションの分数と単位サフィックスが付いた一連の 10 進数として表されます。例: 1h は 1 時間を表し、30m は 30 分を表し、1.5h または 1h30m は 1 時間 30 分を表します。

さらに: 

- `osm.caBundleSecretName` - この文字列は、サービス メッシュ ルート証明書が格納される Kubernetes シークレットの名前です。 Vault を使用する場合 (Tresor とは異なり)、ルート キーはこのシークレットにエクスポートされません。

#### Hashi Vault のインストール

Hashi Vault のインストールは、Open Service Mesh プロジェクトの範囲外です。 通常、これは専任のセキュリティ チームの責任です。 Vault を安全にデプロイして高可用性を実現する方法に関するドキュメントは、[Vault's website](https://learn.hashicorp.com/vault/getting-started/install) で入手できます。

このリポジトリには [script (deploy-vault.sh)](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/demo/deploy-vault.sh) が含まれています。継続的な統合のために Hashi Vault の展開を自動化するために使用されます。 これは厳密に開発目的のみです。 スクリプトを実行すると、[.env](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/.env.example) ファイル。 このスクリプトは、デモンストレーション目的で使用できます。 次の環境変数が必要です。
```bash
export K8S_NAMESPACE=osm-system-ns
export VAULT_TOKEN=xyz
```

`./demo/deploy-vault.sh` スクリプトを実行すると、dev Vault がインストールされます。

```console
NAMESPACE         NAME                                    READY   STATUS    RESTARTS   AGE
osm-system-ns     vault-5f678c4cc5-9wchj                  1/1     Running   0          28s
```

ポッドのログを取得すると、Vault のインストールに関する詳細が表示されます。

```console
==> Vault server configuration:

             Api Address: http://0.0.0.0:8200
                     Cgo: disabled
         Cluster Address: https://0.0.0.0:8201
              Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.4.0

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://0.0.0.0:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: cZzYxUaJaN10sa2UrPu7akLoyU6rKSXMcRt5dbIKlZ0=
Root Token: xyz

Development mode should NOT be used in production installations!

==> Vault server started! Log data will stream in below:
...
```

システムに Vault を展開すると、URL とトークンが生成されます。 たとえば、Vault の URL は「http://vault.<osm-namespace>.svc.cluster.local」で、トークンは「xxx」です。
> 注: `<osm-namespace>` は、osm コントロール プレーンがインストールされている名前空間を指します。

#### Vault で osm-edge を構成する

Vault のインストール後、Helm を使用して osm-edge を展開する前に、Helm チャートで次のパラメーターを指定する必要があります。

```bash
CERT_MANAGER=vault
VAULT_HOST="vault.${K8S_NAMESPACE}.svc.cluster.local"
VAULT_PROTOCOL=http
VAULT_TOKEN=xyz
VAULT_ROLE=openservicemesh
```

ローカル ワークステーションで osm-edge を実行する場合は、次の「osm install」設定オプションを使用します。

```bash
--set osm.certificateProvider.kind="vault"
--set osm.vault.host="localhost"  # or the host where Vault is installed
--set osm.vault.protocol="http"
--set osm.vault.token="xyz"
--set osm.vault.role="openservicemesh'
--set osm.serviceCertValidityDuration=24h
```

#### osm-edge と Vault の統合方法

osm-edge コントロール プレーンが起動すると、新しい証明書発行者がインスタンス化されます。
証明書発行者の種類は、`osm.certificateProvider.kind` セット オプションによって決まります。
これが「vault」に設定されている場合、osm-edge は Vault 証明書発行者を使用します。
これは、`certificate.Manager` を満たす Hashicorp Vault クライアントです。
インターフェース。 次のメソッドを提供します。

```console
  - IssueCertificate - issues new certificates
  - GetCertificate - retrieves a certificate given its Common Name (CN)
  - RotateCertificate - rotates expiring certificates
  - GetAnnouncementsChannel - returns a channel, which is used to announce when certificates have been issued or rotated
```

osm-edge は、Vault サーバー上に CA が既に作成されていることを前提としています。
osm-edge には、専用の Vault ロールも必要です (たとえば、「pki/roles/openservicemesh」)。
`./demo/deploy-vault.sh` スクリプトによって作成された Vault ロールは、次の構成を適用します。これは、開発目的にのみ適しています。

- `allow_any_name`: `true`
- `allow_subdomains`: `true`
- `allow_baredomains`: `true`
- `allow_localhost`: `true`
- `max_ttl`: `24h`

Hashi Vault のサイトには優れた [documentation](https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine) があります。
新しい CA を作成する方法について説明します。 `./demo/deploy-vault.sh` スクリプトは、
次のコマンドを使用して、開発環境をセットアップします。

    export VAULT_TOKEN="xyz"
    export VAULT_ADDR="http://localhost:8200"
    export VAULT_ROLE="openservicemesh

    # Launch the Vault server in dev mode
    vault server -dev -dev-listen-address=0.0.0.0:8200 -dev-root-token-id=${VAULT_TOKEN}

    # Also save the token locally so this is available
    echo $VAULT_TOKEN>~/.vault-token;

    # Enable the PKI secrets engine (See: https://www.vaultproject.io/docs/secrets/pki#pki-secrets-engine)
    vault secrets enable pki;

    # Set the max lease TTL to a decade
    vault secrets tune -max-lease-ttl=87600h pki;

    # Set URL configuration (See: https://www.vaultproject.io/docs/secrets/pki#set-url-configuration)
    vault write pki/config/urls issuing_certificates='http://127.0.0.1:8200/v1/pki/ca' crl_distribution_points='http://127.0.0.1:8200/v1/pki/crl';

    # Configure a role named "openservicemesh" (See: https://www.vaultproject.io/docs/secrets/pki#configure-a-role)
    vault write pki/roles/${VAULT_ROLE} allow_any_name=true allow_subdomains=true;

    # Create a root certificate named "osm.root" (See: https://www.vaultproject.io/docs/secrets/pki#setup)
    vault write pki/root/generate/internal common_name='osm.root' ttl='87600h'

osm-edge コントロール プレーンは、Vault のインストールで行われた操作に関する詳細なログを提供します。

### 証明書マネージャーの使用

[cert-manager](https://cert-manager.io) は、署名付き発行のもう 1 つのプロバイダーです。
証明書を osm-edge サービス メッシュに送信します。秘密鍵を保存する必要はありません。
Kubernetes で。 cert-manager は複数の発行者バックエンドをサポートしています
[core](https://cert-manager.io/docs/configuration/) を cert-manager に、および
プラグ可能な [外部](https://cert-manager.io/docs/configuration/external/)
発行者。

[ACME 証明書](https://cert-manager.io/docs/configuration/acme/)
サービス メッシュ証明書の発行者としてサポートされていません。

osm-edge が証明書を要求すると、cert-manager が作成されます
[`CertificateRequest`](https://cert-manager.io/docs/concepts/certificaterequest/)
構成された発行者によって署名されたリソース。

### osm-edge 署名用に cert-manager を構成する

osm-edge をインストールする前に、cert-manager を最初にインストールし、発行者を準備する必要があります。
証明書プロバイダーとして cert-manager を使用してインストールされます。 あなたは見つけることができます
cert-manager のインストール ドキュメント
[こちら](https://cert-manager.io/docs/installation/)。

cert-manager がインストールされたら、[発行者リソース](https://cert-manager.io/docs/configuration/) を構成して証明書を提供します
リクエスト。 `Issuer` リソースの種類を使用することをお勧めします (
`ClusterIssuer`) は osm-edge 名前空間 (`osm-system` by
デフォルト）。

準備ができたら、発行者のルート CA 証明書を保存することが **必須** です。
osm-edge 名前空間 (デフォルトでは「osm-system」) の Kubernetes シークレットとして
「ca.crt」キー。 ターゲット CA シークレット名は、osm-edge で以下を使用して構成できます。
`osm install --set osm.caBundleSecretName=my-secret-name` (通常は `osm-ca-bundle`)。

```bash
kubectl create secret -n osm-system generic osm-ca-bundle --from-file ca.crt
```

詳細については、[cert-manager デモ](docs/demos/cert-manager_integration) を参照してください。

#### osm-edge が構成された発行者で cert-manager を使用するには、
`osm install` コマンドの次の CLI 引数:

- `--set osm.certificateProvider.kind="cert-manager"` - cert-manager をプロバイダーとして使用するために必要です。
- `--set osm.certmanager.issuerName` - [Cluster]Issuer リソースの名前 (デフォルトは `osm-ca`)。
- `--set osm.certmanager.issuerKind` - 発行者の種類 (`Issuer` または `ClusterIssuer` のいずれか、デフォルトは `Issuer`)。
- `--set osm.certmanager.issuerGroup` - 発行者が属するグループ (すべての主要な発行者タイプである `cert-manager.io` にデフォルト設定)。
