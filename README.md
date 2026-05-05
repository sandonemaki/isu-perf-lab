# isu-perf-lab

## 学習終了時（課金を止める）

```bash
make aws.down       # EC2インスタンス（web・bench）を停止
docker compose down # ローカルのDockerも停止（任意）
```

`make aws.down` でEC2の2台（webサーバー・ベンチマークサーバー）が停止します。停止状態は課金されません。

---

## 学習開始時

EC2を再起動すると**IPアドレスが変わる**ので、SSHの設定も更新する必要があります。

```bash
docker compose up -d        # ローカルのDocker起動
make aws.up                 # EC2インスタンスを起動
make aws.setup-ssh-config   # SSH設定を新しいIPで更新
```

| コマンド                      | やること                                              |
| ----------------------------- | ----------------------------------------------------- |
| `make aws.up`               | EC2を起動して自分のIPをセキュリティグループに追加     |
| `make aws.setup-ssh-config` | `.ssh/config` のIPを新しいIPに書き換えてSSH接続確認 |

---

## 現在の状態確認

いつでも以下で確認できます：

```bash
make aws.status
```

EC2の状態（running /

---
## sqlを作成した場合

ClickHouseが起動中なら一度停止 →  `docker compose down -v`（-v でボリュームを削除して初期化処理を再実行させる）
再起動 → `docker compose up -d`
テーブル確認 → `make perf.open-ch` でClickHouseのWeb UIを開き、`show tables;` を実行して nginx_access_runs が表示されれば成功

## ディレクトリ構造

```
isu-perf-lab/
│
├── 📄 Makefile                    ← よく使うコマンドをまとめた「コマンド集」
├── 📄 compose.yaml                ← ローカルで動くDocker環境の定義（ClickHouse・Grafana）
├── 📄 compose.tool.yaml           ← ClickHouseクライアント専用の使い捨てDocker定義
├── 📄 .envrc                      ← 環境変数の設定ファイル（direnvで自動読み込み）
├── 📄 .envrc.override             ← 個人設定の上書き用（GitHubユーザー名など）
├── 📄 private-isu.yaml            ← AWSのEC2インスタンスを作るCloudFormationの設定
│
├── 📁 makefiles/                  ← Makefileを機能別に分割したファイル群
│   ├── aws.mk                     ←   AWSのEC2起動・停止・SSH設定
│   ├── perf.mk                    ←   ブラウザでGrafana/ClickHouseを開くコマンド
│   ├── ssh.mk                     ←   SSH関連コマンド
│   └── tool.mk                    ←   その他ツール系コマンド
│
├── 📁 scripts/                    ← 実際の処理を行うシェルスクリプト
│   ├── 00-setup.sh                ←   初回セットアップ
│   ├── 01-deploy-app.sh           ←   アプリのデプロイ（webサーバーへ送る）
│   ├── 01-deploy-mysql.sh         ←   MySQLの設定をデプロイ
│   ├── 01-deploy-nginx.sh         ←   Nginxの設定をデプロイ
│   ├── 02-bench.sh                ←   ベンチマーク実行（benchサーバーで動かす）
│   ├── 02-prepare-analyze.sh      ←   ベンチ結果をresults/に保存する準備
│   ├── 04-store-results.sh        ←   results/のJSONをClickHouseに保存
│   └── 99-util.sh                 ←   共通関数（log_info・start_timerなど）
│
├── 📁 etc/                        ← 各サービスの設定ファイル（ローカルDocker用）
│   ├── clickhouse-server/
│   │   └── docker-entrypoint-initdb.d/
│   │       └── 001_results.sql    ←   ClickHouse起動時に自動実行されるSQL（テーブル作成）
│   ├── grafana/provisioning/
│   │   ├── datasources/
│   │   │   └── clickhouse.yaml    ←   GrafanaにClickHouseを接続する設定
│   │   └── dashboards/
│   │       └── default.yaml       ←   Grafanaのダッシュボード読み込み先の設定
│   ├── mysql/                     ←   MySQLの設定（webサーバーにデプロイする）
│   └── nginx/                     ←   Nginxの設定（webサーバーにデプロイする）
│
├── 📁 private_isu/                ← ベンチマーク対象のWebアプリ本体
│   └── webapp/golang/
│       ├── app.go                 ←   GoのWebアプリ（チューニング対象）
│       └── templates/             ←   HTMLテンプレート
│
├── 📁 results/                    ← ベンチマーク結果のJSONが溜まるディレクトリ
│   └── 2026-05-04T08:35:02Z_.../
│       └── result.json            ←   スコア・開始終了時刻が入ったファイル
│
├── 📁 var/lib/grafana/dashboards/ ← Grafanaのダッシュボード定義ファイル
│   └── sample-dashboard-....json  ←   作成したダッシュボードの保存先
│
├── 📁 .ssh/                       ← SSH接続の設定
│   ├── config                     ←   web・benchサーバーのIPとSSH設定
│   └── ssh_config.tmpl            ←   configのテンプレート（aws.setup-ssh-configで生成）
│
└── 📁 .claude/                    ← Claude用（このハンズオンのPDFが入っている）
    └── 実践オブザーバビリティ....pdf
```

### 全体の流れとの対応

```
[AWS EC2]                    [ローカルPC]
  web サーバー  ←デプロイ←  scripts/01-deploy-*.sh
  bench サーバー →ベンチ→   scripts/02-bench.sh
                               ↓
                            results/ にresult.jsonが貯まる
                               ↓
                            scripts/04-store-results.sh
                               ↓
                          [ClickHouse] ← compose.yaml で起動
                               ↓
                           [Grafana]  ← compose.yaml で起動
                               ↓
                          グラフでスコアを可視化
```
