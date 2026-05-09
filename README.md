# isu-perf-lab

## 学習終了時（課金を止める）

```bash
make aws.down       # EC2インスタンス（web・bench・perf）をすべて停止
docker compose down # ローカルのDockerも停止（任意）
```

`make aws.down` でEC2の3台（webサーバー・ベンチマークサーバー・perfサーバー）が停止します。停止状態は課金されません。

---

## 学習開始時

EC2を再起動すると**IPアドレスが変わる**ので、SSHの設定も更新する必要があります。

```bash
docker compose up -d        # ローカルのDocker起動（任意）
make aws.up                 # EC2インスタンス（web・bench・perf）を起動
```

`make aws.up` は perf インスタンスの起動後に `make aws.setup-ssh-config` を自動実行するので、SSH設定の手動更新は不要です。

| コマンド | やること |
| --- | --- |
| `make aws.up` | web・bench・perfのEC2を起動し、自分のIPをセキュリティグループに追加、SSH設定を自動更新 |
| `make aws.setup-ssh-config` | `.ssh/config` のIPを手動で更新したいときに使う |

---

## デプロイの種類と使いどころ

| コマンド | 何をデプロイするか | いつ使うか |
| --- | --- | --- |
| `make deploy` | Goアプリ・Nginx・MySQL設定 | **チューニング中に毎回**（コードや設定を変えたとき） |
| `make perf.deploy` | ClickHouse・Grafana・OTel Collector Gateway・Pyroscope | **初回セットアップ時** または監視設定を変えたとき |
| `make setup` | apt・Docker・OTel Collector 本体（web と perf 両方） | **サーバー作り直し時のみ**（EC2を再作成したとき） |

### チューニング中の基本サイクル

```
app.go を編集
    ↓
make deploy       # Goアプリをwebサーバーに反映
    ↓
make bench        # ベンチマーク実行（トレース・ログが生成される）
    ↓
make analyze      # ログ解析 → ClickHouseに保存
    ↓
make perf.open    # Grafanaで結果を確認
```

### `otel_traces` にデータを入れるには

Grafanaのトレース系クエリ（`otel_traces` テーブル）にデータが入るのは、**OTel対応済みのGoアプリでベンチマークを実行した後**です。

```bash
make deploy   # OTel対応のGoアプリをデプロイ（初回または app.go 変更後）
make bench    # ベンチ実行 → トレースが otel_traces に保存される
```

`make analyze` は不要です（トレースはGoアプリから直接OTel Collector経由でClickHouseに送られます）。

---

## 通常の作業フロー

```bash
make bench      # ① ログ洗い替え → ベンチマーク実行 → ログダウンロード
make analyze    # ② ログ解析 → ClickHouseに保存
make perf.open  # ③ Grafana・ClickHouse・Pyroscopeをブラウザで開く
```

Grafanaでログ系の集計を表示するには `make bench` → `make analyze` でClickHouseにデータを入れる必要があります。
トレース・メトリクス・プロファイリングは `make bench` 実行中にリアルタイムで自動送信されます。
`make perf.open-ch` はClickHouseのWeb UIを開くだけで、データを作る操作ではありません。

| コマンド | データへの影響 |
| --- | --- |
| `make bench` | ベンチマーク実行・ログ取得（`results/` にファイルが増える） |
| `make analyze` | ログを解析してClickHouseにデータを保存 ← **ここでデータが入る（ログ系）** |
| `make perf.open-ch` | ClickHouse Web UIを開くだけ（データの確認・手動クエリ用） |
| `make perf.open-grafana` | Grafanaをブラウザで開く（ClickHouseにデータがあれば表示される） |
| `make perf.open-pyroscope` | Pyroscopeをブラウザで開く（プロファイリング確認用） |

---

## `make perf.deploy` の接続先について

`make perf.deploy` は以下の優先順位で観測スタック（ClickHouse・Grafana・OTel Collector Gateway・Pyroscope）のデプロイ先を自動判定します。

| 状態 | デプロイ先 |
| --- | --- |
| ローカルのDockerが起動中 | デプロイしない（「ローカルにデプロイ済み」メッセージを表示） |
| `PERF_STACK_ID` が存在（perf EC2あり） | OTel Gateway → **perf インスタンス**、OTel Agent → **web インスタンス** |
| `PERF_STACK_ID` が未設定（perf EC2なし） | OTel Gateway → **web インスタンス** |

`make perf.open` / `make perf.open-grafana` / `make perf.-ch` も同様に接続先を自動判定します。

| 状態 | 接続先 |
| --- | --- |
| ローカルのDockerが起動中 | `localhost` |
| perf EC2あり（`PERF_STACK_ID` が存在） | perf インスタンスのIP |
| perf EC2なし | web インスタンスのIP |

---

## `make perf.deploy` でローカル起動中のメッセージが出た場合

`make perf.deploy` を実行したときに以下のメッセージが出た場合：

```
パフォーマンス計測環境はローカルにデプロイされています
```

ローカルのDockerで ClickHouse や Grafana がすでに起動しています。
perf インスタンスまたは web インスタンスにデプロイしたい場合は、ローカルのコンテナを停止してから再実行してください。

```bash
docker compose down   # ローカルのコンテナを停止
make perf.deploy      # perf/webインスタンスにデプロイ
```

---

## Grafanaダッシュボードを更新する場合

### 正しい手順

1. ローカルのダッシュボードファイルを直接編集する

```
var/lib/grafana/dashboards/sample-dashboad-1778073198264.json
```

2. EC2 に反映する

```bash
make perf.deploy
```

`updateIntervalSeconds: 10` の設定により、rsync 後10秒以内に Grafana が自動反映します。

### やってはいけないこと

| NG操作 | 理由 |
|--------|------|
| Grafana UI 上で「Save dashboard（別名保存）」 | 新しいファイルが EC2 上に生成され、同じ UID を持つファイルが複数できる |
| EC2 上でファイルを直接編集 | ローカルとの差分が生まれ、次回 `make perf.deploy` で上書きされて消える |

### なぜ UID の重複が問題になるか

Grafana のダッシュボードは `uid` フィールドで識別されます。同じ `uid` を持つファイルが複数ある場合、Grafana はファイルをアルファベット順に処理しますが、バージョンの大小やスキャンタイミングによって**どのファイルが表示されるか不定**になります。

ダッシュボードファイルは `var/lib/grafana/dashboards/` 内の1ファイルだけを維持し、そのファイルを直接編集してください。

---

## SQLを更新した場合（テーブル定義の変更）

`etc/clickhouse-server/docker-entrypoint-initdb.d/` 配下のSQLを変更したときは、
ClickHouseのボリュームを削除して再起動する必要があります。

```bash
docker compose down -v  # コンテナ停止 + ボリューム削除（テーブルがリセットされる）
docker compose up -d    # 再起動（SQLが自動実行されてテーブルが再作成される）
```

確認：
```bash
make perf.open-ch
# ClickHouse Web UIで以下を実行
SHOW TABLES;
```

---

## 現在の状態確認

```bash
make aws.status   # EC2の状態とIPを確認
```

---

## Makefile コマンド一覧

```bash
make help   # 全コマンドの一覧を表示
```

### よく使うコマンド

**チューニング作業**

| コマンド | 内容 |
| --- | --- |
| `make bench` | ログ洗い替え → ベンチマーク → ログダウンロード（一括） |
| `make analyze` | ログ解析 → ClickHouseへ保存（一括） |
| `make deploy` | アプリ・Nginx・MySQL設定を web インスタンスにデプロイ |
| `make setup` | web・perf 両インスタンスの初回セットアップ（並列実行） |
| `make perf.deploy` | ClickHouse・Grafana・OTel Gateway・Pyroscope をデプロイ |

**Grafana / ClickHouse / Pyroscope を開く**

| コマンド | 内容 |
| --- | --- |
| `make perf.open` | Grafana・ClickHouse・Pyroscope をブラウザで開く |
| `make perf.open-grafana` | Grafana をブラウザで開く |
| `make perf.open-ch` | ClickHouse Web UI をブラウザで開く |
| `make perf.open-pyroscope` | Pyroscope をブラウザで開く |

**SSH 接続**

| コマンド | 内容 |
| --- | --- |
| `make ssh.web` | web インスタンスに SSH 接続 |
| `make ssh.bench` | bench インスタンスに SSH 接続 |
| `make ssh.perf` | perf インスタンスに SSH 接続 |

**AWS インスタンス操作**

| コマンド | 内容 |
| --- | --- |
| `make aws.up` | web・bench・perf を全台起動（SSH設定も自動更新） |
| `make aws.up-web` | web インスタンスのみ起動 |
| `make aws.up-bench` | bench インスタンスのみ起動 |
| `make aws.up-perf` | perf インスタンスのみ起動（SSH設定も自動更新） |
| `make aws.down` | web・bench・perf を全台停止 |
| `make aws.down-web` | web インスタンスのみ停止 |
| `make aws.down-bench` | bench インスタンスのみ停止 |
| `make aws.down-perf` | perf インスタンスのみ停止 |
| `make aws.status` | EC2の状態・IPと CFn スタック一覧を確認 |
| `make aws.setup-ssh-config` | `.ssh/config` のIPを手動更新 |
| `make aws.add-myip-inbound-rule` | 自分のIPをセキュリティグループに追加 |

**CloudFormation スタック管理**

| コマンド | 内容 |
| --- | --- |
| `make aws.create-cfn` | web・bench 用 CFn スタックを作成 |
| `make aws.delete-cfn` | web・bench 用 CFn スタックを削除 |
| `make aws.create-perf-cfn` | perf 用 CFn スタックを作成 |
| `make aws.delete-perf-cfn` | perf 用 CFn スタックを削除 |

**その他**

| コマンド | 内容 |
| --- | --- |
| `make tool.check-required` | 必要コマンドの存在確認（go・docker・aws 等） |

---

## ディレクトリ構造

```
isu-perf-lab/
│
├── Makefile                       ← よく使うコマンドをまとめた「コマンド集」
├── compose.yaml                   ← ローカルDocker環境（ClickHouse・Grafana・Pyroscope）
├── compose.tool.yaml              ← 分析ツール用Docker（alp・pt・ch クライアント）
├── alp.yaml                       ← alpの設定ファイル（Nginxログ解析ルール）
├── .envrc                         ← 環境変数の設定（direnvで自動読み込み）
├── .envrc.override                ← 個人設定の上書き用（GitHubユーザー名など）
├── private-isu.yaml               ← web・bench 用 CloudFormation テンプレート
├── perf.yaml                      ← perf インスタンス用 CloudFormation テンプレート
├── schema.sql                     ← MySQLのテーブル定義
├── dummy-mysql-slow.log.tmpl      ← MySQLスロークエリログのダミーテンプレート
│
├── makefiles/                     ← Makefileを機能別に分割したファイル群
│   ├── aws.mk                     ←   AWSのEC2起動・停止・SSH設定・CFnスタック管理
│   ├── perf.mk                    ←   観測スタックのデプロイ・ブラウザで開く
│   ├── ssh.mk                     ←   SSH接続コマンド（web・bench・perf）
│   └── tool.mk                    ←   必要ツールの確認コマンド
│
├── scripts/                       ← 実際の処理を行うシェルスクリプト
│   ├── 00-setup.sh                ←   インスタンスの初回セットアップ（web・perf 共用）
│   ├── 01-deploy-app.sh           ←   Goアプリのデプロイ
│   ├── 01-deploy-mysql.sh         ←   MySQL設定のデプロイ
│   ├── 01-deploy-nginx.sh         ←   Nginx設定のデプロイ
│   ├── 02-bench.sh                ←   ベンチマーク実行
│   ├── 02-log-rotate.sh           ←   ログの洗い替えと再起動
│   ├── 02-prepare-analyze.sh      ←   ログをresults/にダウンロード
│   ├── 03-analyze.sh              ←   ログ解析（alp・pt-query-digest）→ TSV生成
│   ├── 04-store-nginx-access-runs.sh ← NginxログのTSVをClickHouseに保存
│   ├── 04-store-results.sh        ←   ベンチスコアをClickHouseに保存
│   ├── 04-store-slow-queries.sh   ←   スロークエリのTSVをClickHouseに保存
│   ├── 05-deploy-perf-agent.sh    ←   OTel Collector Agent を web インスタンスにデプロイ
│   ├── 05-deploy-perf-gateway.sh  ←   OTel Collector Gateway を perf/web にデプロイ
│   └── 99-util.sh                 ←   共通関数（log_info・start_timerなど）
│
├── etc/                           ← 各サービスの設定ファイル
│   ├── clickhouse-server/
│   │   └── docker-entrypoint-initdb.d/
│   │       ├── 001_results.sql    ←   resultsテーブル（ベンチスコア）
│   │       ├── 002_nginx_access_logs.sql ← nginx_access_runsテーブル
│   │       └── 003_slow_query_logs.sql   ← slow_runs・slow_queriesテーブル
│   ├── grafana/provisioning/
│   │   ├── datasources/
│   │   │   ├── clickhouse.yaml    ←   GrafanaとClickHouseの接続設定
│   │   │   └── pyroscope.yaml     ←   GrafanaとPyroscopeの接続設定
│   │   └── dashboards/
│   │       └── default.yaml       ←   Grafanaのダッシュボード読み込み先
│   ├── mysql/
│   │   └── mysql.conf.d/
│   │       └── mysqld.cnf         ←   MySQL設定（スロークエリログ有効化など）
│   ├── nginx/
│   │   ├── nginx.conf             ←   Nginx設定
│   │   └── sites-available/
│   │       └── isucon.conf        ←   バーチャルホスト設定
│   ├── otelcol-contrib/
│   │   ├── agent-config.yaml      ←   OTel Collector Agent 設定（web インスタンス用）
│   │   └── gateway-config.yaml    ←   OTel Collector Gateway 設定（perf インスタンス用）
│   └── pyroscope/
│       └── config.yaml            ←   Pyroscope 設定
│
├── dockerfiles/
│   └── Dockerfile.alp             ← alp（Nginxログ解析ツール）のDockerイメージ
│
├── private_isu/                   ← ベンチマーク対象のWebアプリ本体
│   └── webapp/golang/
│       ├── app.go                 ←   GoのWebアプリ（チューニング対象）
│       ├── Dockerfile             ←   Goアプリのコンテナイメージ定義
│       ├── Makefile               ←   ビルド用コマンド
│       ├── setup.sh               ←   web インスタンス上のGoアプリセットアップ
│       └── templates/             ←   HTMLテンプレート
│
├── results/                       ← ベンチマーク結果が溜まるディレクトリ
│   └── 2026-05-06T05:31:52Z_score:3981/
│       ├── result.json            ←   スコア・開始終了時刻
│       ├── var/log/nginx/access.log  ← Nginxアクセスログ
│       ├── var/log/mysql/mysql-slow.log ← MySQLスロークエリログ
│       ├── analyzed_nginx_access.tsv  ← alp解析結果
│       ├── analyzed_slowquery         ← pt-query-digest解析結果（テキスト）
│       ├── analyzed_slowquery.json    ← pt-query-digest解析結果（JSON）
│       ├── analyzed_slow_run.tsv      ← slow_runs用TSV
│       └── analyzed_slow_queries.tsv  ← slow_queries用TSV
│
├── var/lib/grafana/dashboards/    ← Grafanaダッシュボード定義ファイルの保存先
│
└── .ssh/
    ├── config                     ← web・bench・perf の SSH設定（自動生成）
    └── ssh_config.tmpl            ← SSH設定のテンプレート
```

---

## 全体のデータフロー

```
【リアルタイム送信】Goアプリ起動中は常に自動送信

[web インスタンス]
  Goアプリ (isu-go)
    │
    ├─ OTLP HTTP ──→ OTel Collector Agent (:4318)
    │                  │ hostmetrics も収集
    │                  └─ OTLP HTTP ──→ [perf インスタンス]
    │                                     OTel Collector Gateway (:4318)
    │                                       └─ OTLP ──→ ClickHouse
    │                                                     ├── otel_traces    ← トレース
    │                                                     └── otel_metrics_* ← メトリクス
    │
    └─ HTTP (:4040) ─────────────────────→ [perf インスタンス]
                                             Pyroscope (:4040)
                                               └── 継続的プロファイリング


【make bench 実行時】ベンチマーク → ログ取得

[web インスタンス]
  MySQL  → /var/log/mysql/mysql-slow.log
  Nginx  → /var/log/nginx/access.log
       |
       | make bench
       | (02-log-rotate.sh → 02-bench.sh → 02-prepare-analyze.sh)
       ↓
[ローカルPC] results/<タイムスタンプ>_score:<スコア>/
  ├── result.json
  ├── var/log/nginx/access.log
  └── var/log/mysql/mysql-slow.log
       |
       | make analyze  (03-analyze.sh)
       | alp              → analyzed_nginx_access.tsv
       | pt-query-digest  → analyzed_slowquery / analyzed_slowquery.json
       |                  → analyzed_slow_run.tsv / analyzed_slow_queries.tsv
       ↓
[perf インスタンス / ローカルDocker]
  ClickHouse
    ├── results           ← ベンチスコア
    ├── nginx_access_runs ← Nginxアクセスログ統計
    ├── slow_runs         ← スロークエリ全体統計
    └── slow_queries      ← クエリごとの統計


【Grafana で一元可視化】

[perf インスタンス / ローカルDocker]
  Grafana (:3000)
    ├─ ClickHouse ← トレース・メトリクス・ベンチスコア・ログ統計
    └─ Pyroscope  ← 継続的プロファイリング
```

---

## perf インスタンスと web インスタンスの関係

```
┌───────────────────────────────────────────────────────────────────┐
│  web インスタンス                                                   │
│                                                                   │
│  ┌──────────────────────────────┐                                 │
│  │  Goアプリ (isu-go :8080)     │──── プロファイリング(HTTP) ──┐  │
│  │  + pyroscope-go SDK          │                             │  │
│  └──────────────┬───────────────┘                             │  │
│                 │ OTLP HTTP                                    │  │
│                 ▼                                             │  │
│  ┌──────────────────────────────┐                             │  │
│  │  OTel Collector Agent        │                             │  │
│  │  :4317/:4318                 │                             │  │
│  │  ・traces 受信               │                             │  │
│  │  ・hostmetrics 収集          │                             │  │
│  └──────────────┬───────────────┘                             │  │
│                 │ OTLP HTTP (traces + metrics)                 │  │
└─────────────────┼─────────────────────────────────────────────┼──┘
                  │                                             │
                  ▼                                             ▼
┌───────────────────────────────────────────────────────────────────┐
│  perf インスタンス (192.168.1.30)                                   │
│                                                                   │
│  ┌──────────────────────────┐   ┌──────────────────────────────┐  │
│  │  OTel Collector Gateway  │   │  Pyroscope :4040             │  │
│  │  :4318                   │   │  (継続的プロファイリング受信)  │  │
│  │  ・traces + metrics 受信 │   └──────────────┬───────────────┘  │
│  └──────────────┬───────────┘                  │                  │
│                 │ OTLP                          │                  │
│                 ▼                               │                  │
│  ┌──────────────────────────────────────────┐   │                  │
│  │  ClickHouse :9000/:8123                  │   │                  │
│  │  otel_traces         ← トレース          │   │                  │
│  │  otel_metrics_*      ← メトリクス        │   │                  │
│  │  results             ← ベンチスコア      │   │                  │
│  │  nginx_access_runs   ← Nginxログ         │◀──┼── make analyze   │
│  │  slow_runs / slow_queries ← スロークエリ │   │   (ローカルPC)   │
│  └──────────────┬───────────────────────────┘   │                  │
│                 │                                │                  │
│                 ▼                                │                  │
│  ┌──────────────────────────────────────────┐   │                  │
│  │  Grafana :3000                           │◀──┘                  │
│  │  トレース・メトリクス・プロファイリング    │                      │
│  │  ・ベンチスコア等を一元可視化             │                      │
│  └──────────────────────────────────────────┘                      │
└───────────────────────────────────────────────────────────────────┘
```

### データ種別ごとの経路

| データ種別 | 送信元 | 経路 | 保存先 |
|---|---|---|---|
| トレース | Goアプリ | OTLP HTTP → OTel Agent → OTel Gateway | ClickHouse (`otel_traces`) |
| メトリクス (webEC2) | OTel Agent (hostmetrics) | OTLP HTTP → OTel Gateway | ClickHouse (`otel_metrics_*`) |
| プロファイリング | Goアプリ (pyroscope-go SDK) | HTTP → Pyroscope :4040 | Pyroscope |
| Nginxログ | Nginx `access.log` | `make bench` → `make analyze` (TSV) | ClickHouse (`nginx_access_runs`) |
| スロークエリログ | MySQL `slow.log` | `make bench` → `make analyze` (TSV) | ClickHouse (`slow_runs`, `slow_queries`) |
| ベンチスコア | ベンチマーク `result.json` | `make analyze` | ClickHouse (`results`) |

### ポイント

- **トレース・メトリクス** は Goアプリ起動後にリアルタイムで自動送信される（`make bench` 不要）
- **プロファイリング** は起動時に Pyroscope へ接続確認し、応答があれば継続的に送信し続ける
- **ログ系** は `make bench` でログ取得 → `make analyze` で解析・保存の2ステップが必要
