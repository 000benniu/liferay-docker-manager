# Liferay Docker Manager (LDM) クイックスタートガイド

本ガイドは、Windows + PowerShell 7 環境において、Liferay Docker Manager (LDM) を用いてLiferay DXP/Portalのローカル環境を迅速かつクリーンに構築・管理するための手順をまとめています。

---

## 目次
1. [前提条件](#1-前提条件)
2. [LDMのインストールとパス設定](#2-ldmのインストールとパス設定)
3. [推奨されるワークスペースのフォルダ構成](#3-推奨されるワークスペースのフォルダ構成)
4. [初期設定とツールの準備（手動/自動の切り分け）](#4-初期設定とツールの準備)
5. [新規プロジェクトの作成と起動 (Sample.LdmDemo.jp)](#5-新規プロジェクトの作成と起動-sampleldmdemojp)
6. [PaaSバックアップからのリストア (Hydrate機能)](#6-paasバックアップからのリストア-hydrate機能)
7. [スナップショットの取得と復元](#7-スナップショットの取得と復元)
8. [Elasticsearchの設定](#8-elasticsearchの設定)
9. [トラブルシューティング](#9-トラブルシューティング)

---

## 1. 前提条件

本ガイドは以下の環境を前提としています。

* **OS**: Windows 10 または Windows 11
* **シェル**: PowerShell 7 (`pwsh.exe`) ※文字化けを防ぐため、標準のWindows PowerShell(v5)ではなく最新のPowerShell 7およびWindows Terminalの利用を強く推奨します。
* **Docker Desktop**: WSL2バックエンド推奨。
  * **⚠️必須リソース設定**: LiferayとElasticsearchを安定して稼働させるため、Docker Desktopの設定（Settings > Resources）で **CPU: 4コア以上、メモリ: 8GB以上** を割り当ててください。
* **Git**: バージョン管理およびリポジトリ取得用。

---

## 2. LDMのインストールとパス設定

LDMの利用には、用途に合わせて「バイナリ版」と「リポジトリ（ソース）版」の2つの方法があります。ここではご要望に合わせて2パターンの手順を記載します。

### パターンA: スタンドアロンバイナリを利用する（公式推奨）
実行ファイル1つで完結するため、Python環境の構築が不要です。

```powershell
# 1. 保存用フォルダの作成
New-Item -ItemType Directory -Force -Path "$HOME\bin"

# 2. 最新のexeをダウンロード
Invoke-WebRequest -Uri "https://github.com/liferay/liferay-docker-manager/releases/latest/download/ldm-windows.exe" -OutFile "$HOME\bin\ldm.exe"

# 3. ユーザーの環境変数PATHに追加（初回のみ）
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";$HOME\bin", "User")

# 4. 新しいPowerShellウィンドウを開き直して確認
ldm --version
```

### パターンB: リポジトリをダウンロードして `ldm.bat` を利用する
スクリプトを直接確認したり、最新の開発版を利用したい場合の手順です（Python 3.10+ が必要です）。

```powershell
# 1. ツール用フォルダへクローン（C:\path\to\your\workspace は実際のパスに置き換えてください）
cd C:\path\to\your\workspace
git clone https://github.com/liferay/liferay-docker-manager.git
cd liferay-docker-manager

# 2. Python依存関係のインストール
pip install -r requirements.txt

# 3. 環境変数PATHにクローン先のフォルダを追加
# （C:\path\to\your\workspace は実際のパスに置き換えてください）
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";C:\path\to\your\workspace\liferay-docker-manager", "User")

# 4. 新しいPowerShellウィンドウを開き直して確認
ldm --version
```

---

## 3. 推奨されるワークスペースのフォルダ構成

ツール自体（LDM）、実動するLiferayプロジェクト、バックアップを明確に分離するため、以下のようなディレクトリ構成を推奨します。プロジェクトをLDMツールのフォルダ内に直接作成しないことで、ツールのGit管理がクリーンに保たれます。

```text
C:\path\to\your\workspace\
│
├── liferay-docker-manager\  # LDMツールのソースコード（パターンBの場合）
│
├── projects\                # 実動するLiferayプロジェクト群（実行場所）
│   └── Sample.LdmDemo.jp\   # （本ガイドで作成するプロジェクトフォルダ）
│       ├── data\            # マウントされるデータ（DBファイル等）
│       ├── osgi\            # モジュールや設定ファイル
│       └── snapshots\       # ✨ ldm snapshot で取得したスナップショットの保存先
│
└── backups\                 # PaaS環境からダウンロードしたバックアップや初期データの置き場
    └── prod_20260604\       # 展開前の database.gz と volume.tgz ファイルを含むフォルダ
```

---

## 4. 初期設定とツールの準備

LDMを動かすにあたり、必要なツールの準備と責務の切り分け（手動/自動）は以下の通りです。

### 手動でのインストールが必要なもの
ローカル環境で「保護されていない通信（Not Secure）」警告を出さずにHTTPS接続するため、`mkcert` と `openssl` が必要です。Scoopを利用してインストールします。

1. **Scoopとツールのインストール** (管理者権限のPowerShell推奨)
   ```powershell
   # Scoopがない場合はインストール
   Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser; iwr -useb get.scoop.sh | iex
   
   scoop bucket add extras
   scoop install mkcert openssl
   ```
2. **ローカル認証局（CA）のインストール**
   ```powershell
   mkcert -install
   ```
   *※Windowsのセキュリティ警告ダイアログが出たら「はい」を選択してください。*

### LDMが自動で行うもの
* **Traefik SSLプロキシの構築**: `ldm run` 実行時に、HTTPS通信を仲介するグローバルプロキシコンテナが自動で立ち上がります。
* **SSL証明書の自動生成**: プロジェクト起動時、指定されたドメイン（例: Sample.LdmDemo.jp）に対してLDMが内部で `mkcert` を呼び出し、Traefik用の証明書を動的に生成・配置します。
* **Docker Composeの生成と管理**: データベース（MySQL等）とLiferayコンテナの接続設定や、ボリュームのマウント設定をすべて自動で最適化し `docker-compose.yml` を生成します。

**システムの健康状態チェック:**
以下のコマンドで、必要なツールやリソースが揃っているか診断できます。
```powershell
ldm system doctor
```
*※旧コマンド `ldm doctor` も引き続き使用できます。*

---

## 5. 新規プロジェクトの作成と起動 (Sample.LdmDemo.jp)

ローカルに完全にクリーンな新規環境を構築する手順です。

1. **プロジェクトの初期化**
   プロジェクト群を管理するフォルダに移動し、カスタムドメインを指定して初期化します。
   ```powershell
   cd C:\path\to\your\workspace\projects
   ldm init Sample.LdmDemo.jp --host-name Sample.LdmDemo.jp
   ```
2. **名前解決（hostsファイル）の修正**
   カスタムドメイン（`Sample.LdmDemo.jp`）がローカルマシン（`127.0.0.1`）を向くようにhostsファイルを修正します。
   ```powershell
   ldm system fix-hosts Sample.LdmDemo.jp
   ```
   *※実行時にWindowsの権限昇格プロンプト（UAC）が表示されます。*
3. **プロジェクトの起動**
   ```powershell
   ldm run Sample.LdmDemo.jp
   ```
4. **ブラウザでのアクセス**
   ターミナルに「✅ Liferay is ready!」と表示されたら、ブラウザで **`https://Sample.LdmDemo.jp`** にアクセスします。緑色の鍵マークがついたセキュアな状態で表示されます。

---

## 6. PaaSバックアップからのリストア (Hydrate機能)

Liferay Cloud (PaaS) から取得したバックアップデータ（データベースの `.gz` 形式ダンプと、Document Libraryの `.tgz` 形式アーカイブ）を利用し、完全に同期されたローカル環境を構築する方法です。

1. **バックアップの配置**
   ファイルをワークスペース内のフォルダ（例: `C:\path\to\your\workspace\backups\prod_20260604`）に配置します。フォルダ内には `database.gz` (または `.sql`) と `volume.tgz` が含まれている必要があります。
2. **Hydrateコマンドの実行**
   `init` コマンドの代わりに `hydrate` コマンドを使用し、バックアップフォルダのパスと作成したいプロジェクト名を指定します。
   ```powershell
   cd C:\path\to\your\workspace\projects
   ldm hydrate ..\backups\prod_20260604 Sample.LdmDemo.jp
   ```
   *💡ポイント: LDMは自動的にデータベースの種類（MySQL/PostgreSQL）を判別し、解凍、DBのリストア、Document Libraryの展開をすべて自動で行います。*
3. **名前解決と起動**
   hostsファイルを修正し、`--host-name` でカスタムドメインを指定して起動します。
   ```powershell
   ldm system fix-hosts Sample.LdmDemo.jp
   ldm run Sample.LdmDemo.jp --host-name Sample.LdmDemo.jp
   ```

---

## 7. スナップショットの取得と復元

LDMはプロジェクトの状態（データベース、OSGiステート、Document Library、Elasticsearchのインデックス状態を含む）を「スナップショット」として保存し、いつでも元に戻すことができます。

**※重要:** スナップショットの取得・復元は、プロジェクトが**停止している状態**で行う必要があります。
```powershell
ldm stop Sample.LdmDemo.jp
```

### スナップショットの取得 (Create)
プロジェクトの安定した状態を保存します。
```powershell
# わかりやすい名前（例: "base_install"）をつけて保存
ldm snapshot Sample.LdmDemo.jp --name "base_install"
```
*※スナップショットのデータはプロジェクトフォルダ内の `snapshots/` サブフォルダに保存されます。*

### スナップショットのリストア (Restore)
環境が壊れたり、テスト前に戻したい場合に使用します。
```powershell
# 1. 取得済みスナップショットの一覧を確認
ldm restore Sample.LdmDemo.jp --list

# 2. 指定した名前のスナップショットに完全に巻き戻す
ldm restore Sample.LdmDemo.jp --name "base_install"
```
リストア完了後、`ldm run Sample.LdmDemo.jp` で起動すれば、保存した当時の状態で再開されます。

---

## 8. Elasticsearchの設定

LDMはElasticsearchを2つのモードで提供します。デフォルトは **Sidecarモード** で、追加設定なしにElasticsearchが動作します。

### モードの比較

| モード | 動作方式 | 特徴 |
|---|---|---|
| **Sidecar（デフォルト）** | Liferayコンテナ内に組み込み | 設定不要、単一プロジェクト向け |
| **Shared（グローバル）** | 専用コンテナ `liferay-search-global` | 複数プロジェクトで共有、本番環境に近い |

### Sidecarモード（デフォルト・追加設定不要）

`ldm init` / `ldm run` のデフォルトはSidecarモードです。ElasticsearchはLiferayコンテナ内で自動起動するため、追加の手順は不要です。

起動後、Liferayの管理画面（**コントロールパネル → 検索**）からElasticsearchの接続状況を確認できます。

### Sharedモード（専用Elasticsearchコンテナ）

複数のプロジェクト間でElasticsearchを共有したい場合や、本番環境に近い構成が必要な場合に使用します。

**1. 初回セットアップ（グローバルElasticsearchコンテナの起動）**
```powershell
ldm infra setup --search
```

**2. 既存プロジェクトをSidecar → Sharedに移行**
```powershell
ldm stop Sample.LdmDemo.jp
ldm infra migrate-search Sample.LdmDemo.jp
ldm run Sample.LdmDemo.jp
```

*💡ポイント: SharedモードではElasticsearchのインデックスにプロジェクト名プレフィックス（例: `ldm-Sample.LdmDemo.jp-`）が自動付与されるため、複数プロジェクトのデータが混在することはありません。*

### Elasticsearch 7の使用

デフォルトはElasticsearch 8です。ES7が必要な場合は `--es7` フラグを追加します。

```powershell
# グローバルインフラをES7で起動（初回のみ）
ldm infra setup --search --es7

# プロジェクトをES7で起動
ldm run Sample.LdmDemo.jp --es7
```

---

## 9. トラブルシューティング

### 「保護されていない通信 (Not Secure)」 または 「certificate is not trusted」エラーが出る場合
新規作成したプロジェクトにブラウザでアクセスした際、セキュリティ証明書のエラーが表示される場合は、ローカルの証明書（mkcertのルートCA）がOSに信頼されていません。以下の手順で解決します。

1. **ローカル認証局（CA）がOSに信頼されているか確認・再登録する**
   すでに `mkcert` 自体はインストール済みでも、OSのルート証明書ストアへの登録が漏れている（またはセキュリティ警告をキャンセルしてしまった）場合があります。PowerShellを開き、以下のコマンドを再度実行して確実に登録してください。
   ```powershell
   mkcert -install
   ```
   *⚠️重要: Windowsの「セキュリティ警告」ダイアログがポップアップ表示された場合は、必ず **「はい(Y)」** をクリックしてください。ここで許可しないと信頼されません。*

2. **インフラストラクチャと証明書を強制再起動する**
   LDMの証明書管理システム（Traefik）に新しい設定を読み込ませるため、以下の順番でコマンドを実行します。
   ```powershell
   # 1. 念のためプロジェクトを停止
   ldm stop Sample.LdmDemo.jp
   
   # 2. SSL証明書を強制的に再生成
   ldm infra renew-ssl
   
   # 3. LDMのグローバルインフラを再起動
   ldm infra restart
   
   # 4. プロジェクトを再起動
   ldm run Sample.LdmDemo.jp
   ```

3. **ブラウザの再起動**
   証明書のキャッシュをクリアするため、開いているChromeやEdgeのウィンドウを **すべて閉じてから** 、もう一度開き直してアクセスしてください。緑色の鍵マークが表示されれば成功です。
