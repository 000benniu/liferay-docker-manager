# LDM × Cloudflare Tunnel 連携ガイド

## 概要

ファイアウォールのポートを開放することなく、LDMで構築したローカルのLiferay環境をインターネットに安全に公開します。

このガイドでは、ローカル開発時も外部公開時も**同じドメイン（例: `sample.ldmdemo.jp`）を使い続ける**構成を実現します。これにより、ローカルのPCからもスマートフォンなど外部デバイスからも、同一のURLでLiferayにアクセスできます。

> 💡 **費用について:** このガイドで使用するCloudflare Tunnelを含むCloudflare Zero Trustの機能は、**個人・小規模利用の範囲であればすべて無料枠内**で利用できます。

---

## 前提条件

本ガイドを始める前に、以下がすべて完了していることを確認してください。

| 確認項目 | 内容 |
|---|---|
| ✅ ドメイン取得済み | 公開に使用するドメイン（例: `ldmdemo.jp`）を取得済み |
| ✅ Cloudflare登録済み | 取得したドメインがCloudflareに追加され、ネームサーバーの切り替えが完了している |
| ✅ LDM動作確認済み | `ldm run` でローカル環境が起動することを確認済み |

---

## 設定手順

### 手順1. Cloudflare Zero Trust でトンネルを作成し、トークンを取得する

1. [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) にログインし、左メニューの「**Networks**」>「**Tunnels**」へ移動します。
2. 「**Create a tunnel**」をクリックし、タイプとして「**Cloudflared**」を選択して次へ進みます。
3. トンネルに任意の名前を付けます（例: `ldm-local-tunnel`）。
4. 「Select your environment」で「**Docker**」を選択します。
5. 画面に `docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token ey...` のようなコマンドが表示されます。このコマンド内の **`--token` の後の文字列（`ey...` から始まるトークン）だけ** をコピーし、メモ帳などに控えます。

> ⚠️ **重要:** 表示された `docker run ...` コマンドをそのまま実行しないでください。Liferayと同じDockerネットワーク（`liferay-net`）に接続できず、通信できません。**トークン文字列だけをコピーしてください。**

---

### 手順2. `docker-compose.override.yml` を作成して `cloudflared` コンテナを追加する

プロジェクトフォルダ（`my-project/` の直下）に `docker-compose.override.yml` という名前でファイルを新規作成し、以下の内容を貼り付けます。`TUNNEL_TOKEN=` の後に、手順1でコピーしたトークンを貼り付けてください。

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=ey...（手順1でコピーしたトークンをここに貼り付ける）
    networks:
      - liferay-net
    restart: unless-stopped
```

> 💡 **ポイント:** `docker-compose.override.yml` はLDMが自動生成する `docker-compose.yml` を直接編集せず、設定を追加するための仕組みです。LDMをアップデートしてもこのファイルの内容は保持されます。

---

### 手順3. LDMプロジェクトのドメインを公開用ドメインに設定する

ローカル開発時も外部からのアクセス時も、同じドメインで動作するようにLDMを設定します。`my-project` は実際のプロジェクト名に置き換えてください。

```powershell
# プロジェクトを停止する
ldm stop my-project

# プロジェクトフォルダに移動する（実際のパスに合わせてください）
cd C:\path\to\your\workspace\projects\my-project

# ホスト名を公開ドメインに変更する
ldm config set host_name sample.ldmdemo.jp
```

**SSL証明書の再生成**

新しいドメインに対応した証明書を生成します。

> 💡 `mkcert` のインストールとCA登録がまだの場合は、先に以下を実行してください。Windowsのセキュリティ警告が出たら「はい」をクリックしてください。
> ```powershell
> mkcert -install
> ```

```powershell
# 新しいドメインのSSL証明書を強制再生成する
ldm infra renew-ssl

# LDMのグローバルインフラを再起動する
ldm infra restart
```

**hostsファイルへのドメイン登録**

新しいドメイン（`sample.ldmdemo.jp`）がローカルPC（`127.0.0.1`）を向くようにhostsファイルを更新します。

```powershell
ldm system fix-hosts sample.ldmdemo.jp
```

*※実行時にWindowsの権限昇格プロンプト（UAC）が表示されます。「はい」をクリックしてください。*

---

### 手順4. プロジェクトを起動してトンネルを接続する

以下のコマンドでLiferayを起動します。`cloudflared` コンテナも同時に起動し、Cloudflareとのトンネル接続が確立されます。

```powershell
ldm run my-project
```

ターミナルに「✅ Liferay is ready!」と表示されれば起動完了です。

> ⚠️ **重要:** この手順でローカルのトンネルを起動してからでないと、次の手順5（Cloudflare側のルーティング設定）に進めません。`ldm run` が完了してから次に進んでください。

---

### 手順5. Cloudflare Zero Trust でルーティングを設定する

[Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) に戻り、作成したトンネルの設定画面を開いて「**Public Hostname**」タブから転送先を設定します。

1. 「**Add a public hostname**」をクリックします。
2. 以下の通りに入力して保存します。

**Hostname（公開URLの設定）**

| 項目 | 入力値 |
|---|---|
| **Subdomain** | `sample` |
| **Domain** | プルダウンから自分のドメインを選択（例: `ldmdemo.jp`） |
| **Path** | （空欄） |

**Service（転送先の設定）**

| 項目 | 入力値 |
|---|---|
| **Type** | `HTTP` |
| **URL** | `liferay:8080` |

> 💡 **ポイント:** Type は `HTTPS` ではなく `HTTP` を選択してください。Liferayコンテナはポート8080でHTTPを待ち受けており、HTTPS化はCloudflare側で自動処理されます。`liferay` はDockerネットワーク内のサービス名です。

---

## 確認とテスト

設定完了後、以下の2パターンでアクセスを確認してください。

| アクセス元 | URL | 期待結果 |
|---|---|---|
| **PC（ローカル）** | `https://sample.ldmdemo.jp` | hostsファイル経由でTraefikに直接接続 → Liferay表示 |
| **スマートフォン（外部回線）** | `https://sample.ldmdemo.jp` | Cloudflare Tunnel経由でLiferay表示 |

> 💡 どちらの経路からアクセスしても、Liferayは自分が `sample.ldmdemo.jp` であることを認識しているため、リンク切れや別ドメインへのリダイレクトなどのトラブルは発生しません。

---

## 通信の仕組み

```
【外部からのアクセス（スマートフォン等）】
外部デバイス
  → Cloudflare CDN (HTTPS)
  → Cloudflare Tunnel
  → cloudflaredコンテナ (liferay-net)
  → liferayコンテナ (HTTP:8080)

【ローカルからのアクセス（PC）】
PCブラウザ
  → hostsファイル (127.0.0.1)
  → Traefikプロキシ (HTTPS:443)
  → liferayコンテナ (HTTP:8080)
```
