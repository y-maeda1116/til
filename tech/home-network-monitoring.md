## 目的

Mac miniを使って、家庭内ネットワークとPi-holeを監視します。

- ルーター、NAS、外部ネットワークの死活監視
- Ping遅延とパケットロスの可視化
- Pi-holeのDNS問い合わせ数・ブロック率の可視化
- Grafana Cloudでのダッシュボード表示とアラート
- ローカルPrometheus／Grafanaの運用を省略

## 構成

```
家庭内端末
    │ DNS
    ▼
Mac mini
  ├─ Pi-hole
  ├─ Pi-hole Exporter
  ├─ Blackbox Exporter
  └─ Grafana Alloy
          │ Prometheus Remote Write / HTTPS
          ▼
Grafana Cloud
  ├─ Prometheus互換ストレージ
  ├─ Grafanaダッシュボード
  └─ Alerting
```

Grafana Cloudへの通信はMac miniからの外向きHTTPS通信です。インターネット側から自宅へのポート開放は行いません。

---

## 前提条件

- 常時稼働するMac mini
- Docker DesktopまたはOrbStack
- Git
- Grafana Cloudアカウント
- Mac miniのLAN内IPアドレスを固定、またはルーターでDHCP予約
- ルーターのDHCP/DNS設定を変更できること

以下は例としてMac miniを `192.168.1.10`、ルーターを `192.168.1.1` とします。実際の環境に置き換えてください。

## 1. Grafana Cloudを準備する

1. Grafana CloudでStackを作成する。
2. StackのGrafanaへログインする。
3. **Connections**からGrafana AlloyまたはHosted Metricsの接続画面を開く。
4. Prometheus Remote Write用の以下を確認する。
    - Remote Write URL
    - Metrics User ID
    - Access Policy Token
5. トークンにはメトリクス送信に必要な最小限の権限だけを付ける。

認証情報はGitHubへコミットしません。

## 2. リポジトリを作成する

```bash
mkdir home-network-observability
cd home-network-observability
git init
mkdir -p alloy blackbox pihole/etc-pihole pihole/etc-dnsmasq.d
```

最終的な構成です。

```
home-network-observability/
├── .env.example
├── .gitignore
├── compose.yaml
├── README.md
├── alloy/
│   └── config.alloy
├── blackbox/
│   └── blackbox.yml
└── pihole/
    ├── etc-pihole/
    └── etc-dnsmasq.d/
```

## 3. `.gitignore`を作成する

```
.env
.DS_Store

# Pi-holeの実データは環境固有のためコミットしない
pihole/etc-pihole/*
pihole/etc-dnsmasq.d/*

# ディレクトリ自体は維持する
!pihole/etc-pihole/.gitkeep
!pihole/etc-dnsmasq.d/.gitkeep
```

```bash
touch pihole/etc-pihole/.gitkeep
touch pihole/etc-dnsmasq.d/.gitkeep
```

## 4. `.env.example`を作成する

```
TZ=Asia/Tokyo

# Pi-hole管理画面のパスワード
PIHOLE_PASSWORD=change-me

# Grafana Cloud: Connections画面で確認する
GRAFANA_CLOUD_PROMETHEUS_URL=https://prometheus-xxx.grafana.net/api/prom/push
GRAFANA_CLOUD_PROMETHEUS_USER=000000
GRAFANA_CLOUD_API_TOKEN=glc_xxxxxxxxxxxxxxxxxxxx
```

ローカル用ファイルを作成し、実際の値を設定します。

```bash
cp .env.example .env
chmod 600 .env
```

> `.env`は絶対にGitへコミットしないでください。誤ってコミットした場合は、Git履歴から消すだけでなくGrafana Cloudのトークンを失効・再発行します。
> 

## 5. Blackbox Exporterを設定する

`blackbox/blackbox.yml`：

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4

  http_2xx:
    prober: http
    timeout: 10s
    http:
      preferred_ip_protocol: ip4
      follow_redirects: true

  dns_udp:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: ip4
      transport_protocol: udp
      query_name: example.com
      query_type: A
```

## 6. Grafana Alloyを設定する

`alloy/config.alloy`：

```
prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = sys.env("GRAFANA_CLOUD_PROMETHEUS_URL")

    basic_auth {
      username = sys.env("GRAFANA_CLOUD_PROMETHEUS_USER")
      password = sys.env("GRAFANA_CLOUD_API_TOKEN")
    }
  }
}

// Alloy自身の稼働状態
prometheus.scrape "alloy" {
  targets = [
    {
      "__address__" = "127.0.0.1:12345",
      "instance"    = "mac-mini",
      "site"        = "home",
    },
  ]

  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}

// Pi-hole Exporter
prometheus.scrape "pihole" {
  targets = [
    {
      "__address__" = "pihole-exporter:9617",
      "instance"    = "home-pihole",
      "site"        = "home",
    },
  ]

  scrape_interval = "30s"
  forward_to      = [prometheus.remote_write.grafana_cloud.receiver]
}

// 家庭内機器と外部ネットワークのPing監視
prometheus.scrape "blackbox_icmp" {
  targets = [
    {
      "__address__"  = "blackbox-exporter:9115",
      "__param_module" = "icmp",
      "__param_target" = "192.168.1.1",
      "instance"       = "home-router",
      "site"           = "home",
    },
    {
      "__address__"  = "blackbox-exporter:9115",
      "__param_module" = "icmp",
      "__param_target" = "1.1.1.1",
      "instance"       = "cloudflare-dns",
      "site"           = "external",
    },
    {
      "__address__"  = "blackbox-exporter:9115",
      "__param_module" = "icmp",
      "__param_target" = "8.8.8.8",
      "instance"       = "google-dns",
      "site"           = "external",
    },
  ]

  metrics_path    = "/probe"
  scrape_interval = "30s"
  scrape_timeout  = "10s"
  forward_to      = [prometheus.remote_write.grafana_cloud.receiver]
}

// Pi-holeのDNS応答監視
prometheus.scrape "blackbox_dns" {
  targets = [
    {
      "__address__"    = "blackbox-exporter:9115",
      "__param_module" = "dns_udp",
      "__param_target" = "pihole:53",
      "instance"       = "home-pihole-dns",
      "site"           = "home",
    },
  ]

  metrics_path    = "/probe"
  scrape_interval = "30s"
  scrape_timeout  = "10s"
  forward_to      = [prometheus.remote_write.grafana_cloud.receiver]
}
```

NASなどを追加する場合は、`blackbox_icmp`の`targets`に同様のエントリーを追加します。

## 7. Docker Composeを作成する

`compose.yaml`：

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    environment:
      TZ: ${TZ}
      FTLCONF_webserver_api_password: ${PIHOLE_PASSWORD}
      FTLCONF_dns_listeningMode: all
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    volumes:
      - ./pihole/etc-pihole:/etc/pihole
      - ./pihole/etc-dnsmasq.d:/etc/dnsmasq.d

  pihole-exporter:
    image: ekofr/pihole-exporter:latest
    container_name: pihole-exporter
    restart: unless-stopped
    depends_on:
      - pihole
    environment:
      PIHOLE_HOSTNAME: pihole
      PIHOLE_PORT: "80"
      PIHOLE_PASSWORD: ${PIHOLE_PASSWORD}
      PORT: "9617"

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    restart: unless-stopped
    command:
      - --config.file=/etc/blackbox_exporter/config.yml
    cap_add:
      - NET_RAW
    volumes:
      - ./blackbox/blackbox.yml:/etc/blackbox_exporter/config.yml:ro

  alloy:
    image: grafana/alloy:latest
    container_name: grafana-alloy
    restart: unless-stopped
    depends_on:
      - pihole-exporter
      - blackbox-exporter
    env_file:
      - .env
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    ports:
      - "127.0.0.1:12345:12345"
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
      - alloy-data:/var/lib/alloy/data

volumes:
  alloy-data:
```

> イメージを`latest`で検証した後、安定運用では動作確認済みのバージョンへ固定してください。依存イメージの更新は定期的に内容を確認してから反映します。
> 

## 8. 設定を検証して起動する

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

ログを確認します。

```bash
docker compose logs --tail=100 alloy
docker compose logs --tail=100 blackbox-exporter
docker compose logs --tail=100 pihole-exporter
```

管理画面：

- Pi-hole: `http://192.168.1.10:8080/admin/`
- Alloy: Mac mini上で `http://127.0.0.1:12345/`

Exporterを確認します。

```bash
docker compose exec alloy wget -qO- http://pihole-exporter:9617/metrics | head
docker compose exec alloy wget -qO- 'http://blackbox-exporter:9115/probe?module=icmp&target=1.1.1.1' | head
```

## 9. 家庭内DNSをPi-holeへ切り替える

1. Mac miniのLAN内IPを固定、またはルーターでDHCP予約する。
2. 端末1台だけ、DNSサーバーをMac miniのIP `192.168.1.10` に手動設定する。
3. 名前解決とWeb閲覧を確認する。
4. 問題がなければ、ルーターのDHCP設定で配布DNSを `192.168.1.10` に変更する。
5. 家庭内端末のDHCPリースを更新、またはWi-Fiを再接続する。
6. Pi-hole管理画面で問い合わせが記録されることを確認する。

確認コマンド：

```bash
nslookup example.com 192.168.1.10
nslookup doubleclick.net 192.168.1.10
```

### DNS切り替え時の注意

- Mac miniやDockerが停止すると、家庭内DNSも停止します。
- 最初はルーターの設定を元に戻す手順を控えてから変更します。
- セカンダリDNSにパブリックDNSを指定すると、端末がPi-holeを迂回することがあります。
- 高可用性が必要なら、別の端末で2台目のPi-holeを用意します。
- VPN、DNS over HTTPS、Apple Private RelayなどはPi-holeを迂回する場合があります。
- macOSですでに53番ポートが使用されている場合は、競合するサービスを確認します。

## 10. Grafana Cloudで受信を確認する

Grafana CloudのExploreでPrometheus互換データソースを選択し、以下を実行します。

```
up
```

Blackbox監視：

```
probe_success
```

```
probe_duration_seconds
```

パケットロスや到達不能を確認するときは、`instance`ラベルで絞り込みます。

```
probe_success{instance="home-router"}
```

Pi-holeのメトリクス名はExporterのバージョンによって変わる可能性があります。Exploreのメトリクス一覧、または次の出力で確認します。

```bash
docker compose exec alloy wget -qO- http://pihole-exporter:9617/metrics
```

## 11. ダッシュボードを作成する

最初に以下のパネルを作成します。

| パネル | 指標 |
| --- | --- |
| ルーター稼働 | `probe_success{instance="home-router"}` |
| 外部接続 | `probe_success{site="external"}` |
| Ping/DNS応答時間 | `probe_duration_seconds` |
| Pi-hole稼働 | Pi-hole Exporterのstatus系メトリクス |
| DNS問い合わせ数 | Pi-hole Exporterのquery系メトリクス |
| ブロック率 | Pi-hole Exporterのblocked/percentage系メトリクス |

Pi-hole Exporter向けの公開ダッシュボードをインポートする場合は、Grafana Dashboard ID `10176` を出発点にし、現在のメトリクス名に合わせて修正します。

## 12. アラートを設定する

最低限、以下を設定します。

### ルーター到達不能

```
probe_success{instance="home-router"} == 0
```

推奨条件：2～5分継続した場合に通知。

### 外部ネットワーク到達不能

単一の外部IP障害を回線障害と誤認しないよう、`1.1.1.1`と`8.8.8.8`の両方が失敗した場合に通知します。

### Pi-hole停止

Pi-hole Exporterのstatus系メトリクス、またはDNSプローブの次を利用します。

```
probe_success{instance="home-pihole-dns"} == 0
```

通知先はメール、Slack、Discordなどから選択します。

## 13. セキュリティとプライバシー

- Grafana CloudのトークンをGitHubへ保存しない。
- `.env`の権限を`600`にする。
- Grafana Cloudトークンは送信専用・最小権限にする。
- Pi-hole管理画面をインターネットへ公開しない。
- ルーターで53、8080、12345、9115、9617番ポートをポート転送しない。
- Grafana Cloudへ送るのは集計メトリクスだけにし、DNSクエリログやアクセス先ドメインは送信しない。
- 外出先からMac miniを操作する場合はTailscaleなどのVPNを利用する。
- GitHubへpushする前に、秘密情報や家庭内の機器名・IPアドレスを確認する。

確認例：

```bash
git status
git diff --cached
grep -R "glc_" . --exclude=.env --exclude-dir=.git || true
```

## 14. GitHubへ保存する

```bash
git add .
git commit -m "Add home network observability stack"
git branch -M main
git remote add origin git@github.com:YOUR_ACCOUNT/home-network-observability.git
git push -u origin main
```

家庭内IPや構成を公開したくない場合は、GitHubリポジトリをPrivateにします。

## 15. 日常運用

状態確認：

```bash
docker compose ps
docker compose logs --tail=100 alloy
```

再起動：

```bash
docker compose restart
```

更新：

```bash
docker compose pull
docker compose up -d
docker image prune
```

停止：

```bash
docker compose down
```

Pi-holeの設定データを含めて削除しない限り、`docker compose down -v`は実行しません。

## 16. ロールバック

問題が発生した場合：

1. ルーターのDHCP/DNS設定を変更前の値へ戻す。
2. 端末のWi-Fiを再接続してDHCP情報を更新する。
3. パブリックDNSで名前解決できることを確認する。
4. コンテナを停止する。

```bash
docker compose down
```

## 17. 次の拡張候補

- NAS、テレビ、ゲーム機、IoT機器のPing監視
- HTTP/HTTPSサービスの監視
- TelegrafをmacOSへ直接導入し、Mac mini本体のCPU・メモリ・ディスクを監視
- SNMP対応ルーター／スイッチのトラフィック監視
- Tailscale経由の安全なリモート管理
- 2台目のPi-holeによるDNS冗長化
- Lokiへのログ送信。ただしDNSクエリログはプライバシーを考慮して送信しない

## 参考資料
- **Grafana Alloyのインストール** 
  https://grafana.com/docs/alloy/latest/set-up/install/
    
- **Grafana AlloyからPrometheusへメトリクスを送信**    
    https://grafana.com/docs/alloy/latest/tutorials/send-metrics-to-prometheus/
    
- **Alloy `prometheus.remote_write`**    
    https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.remote_write/
    
- **Prometheus Blackbox Exporter**    
    https://github.com/prometheus/blackbox_exporter
    
- **Pi-hole Prometheus Exporter**    
    https://github.com/eko/pihole-exporter
    
- **Pi-hole Exporter Grafana Dashboard（ID: 10176）**    
    https://grafana.com/grafana/dashboards/10176/
