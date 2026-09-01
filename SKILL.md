---
name: aws-gcp-multi
description: >
  AWS と GCP (Google Cloud Platform) のマルチクラウド環境を設計・構築・運用する際のガイドスキル。
  設計観点（ネットワーク接続・ID統合・データ連携）、構築観点（IaC・CI/CD・命名規則）、
  運用観点（監視統合・コスト管理・セキュリティ統制）の考慮点と Hint&Tips を提供する。
  AWS 側は aws-mcp、GCP 側は google-developer-knowledge、構成図は drawio MCP、
  Terraform コードは terraform-aws / terraform-gcp スキルを活用する。
---

# AWS × GCP マルチクラウド 設計・構築・運用ガイド スキル

このスキルは、AWS と GCP (Google Cloud Platform) を併用するマルチクラウド環境において、
**設計**・**構築**・**運用** の各フェーズで必要な考慮点、ベストプラクティス、
Hint&Tips をガイドするものです。

---

## 概要 — マルチクラウドが必要になる典型的シナリオ

| シナリオ | 説明 |
|---|---|
| ベンダーロックイン回避 | 特定クラウドへの依存度を下げ、交渉力・柔軟性を確保する |
| ベスト・オブ・ブリード | AWS の広範なサービス群 + GCP の Data/AI・BigQuery・GKE など各社の強みを活用 |
| 規制・データ主権 | 地域ごとに異なるクラウドプロバイダーを使い分ける |
| M&A・組織統合 | 買収先が別クラウドを使用しており段階的に統合する |
| DR / BCP | 一方のクラウド障害時に他方へフェイルオーバーする |
| データ分析活用 | 複数クラウドのデータを GCP (BigQuery) に集約して分析する |

---

## Step 1 — 要件のヒアリング

照会者に以下を確認する（未確定の項目のみ質問する）。

1. **マルチクラウド採用の目的** — 上表のどのシナリオに該当するか。
2. **対象ワークロード** — AWS 側と GCP 側それぞれで稼働させるサービス・アプリ。
3. **接続要件** — 両クラウド間のネットワーク接続が必要か（帯域・レイテンシ要件）。
4. **ID / 認証統合** — シングルサインオン（SSO）やワークロード認証フェデレーションの要否。
5. **データ連携** — クラウド間でデータを共有・同期する必要があるか。
6. **ガバナンス方針** — タグ / ラベル・命名規則・コスト管理の統一ルール。
7. **運用体制** — 監視・アラート・インシデント対応の統合方針。
8. **IaC / CI/CD** — Terraform 等で両環境を統一管理するか。
9. **セキュリティ基準** — コンプライアンス要件（ISO27001, SOC2, PCI-DSS 等）。
10. **構成図の要否** — AWS / GCP それぞれのアーキテクチャ図が必要か。

---

## Step 2 — AWS 側の情報収集とベストプラクティス確認

### 2-1. AWS ドキュメント・ベストプラクティスの検索

`aws-mcp` の `search_documentation` ツールを使い、対象サービスのベストプラクティスや
マルチクラウド関連のガイダンスを取得する。

```
topics: ["general"]
search_phrase: "<対象サービス> multi-cloud best practices"
```

### 2-2. AWS Well-Architected Framework の観点

以下の柱に沿って AWS 側の設計を評価する：

| 柱 | マルチクラウドでの考慮点 |
|---|---|
| 運用の優秀性 | CloudWatch + Cloud Monitoring (旧 Stackdriver) の統合、IaC の統一 |
| セキュリティ | IAM と Google Cloud IAM のフェデレーション、暗号鍵の管理分離 |
| 信頼性 | クロスクラウド DR、Route 53 / Cloud DNS の連携 |
| パフォーマンス効率 | クラウド間レイテンシの最小化、データ配置最適化 |
| コスト最適化 | エグレス料金の削減、Reserved / Savings Plans / CUD の活用 |
| 持続可能性 | リージョン選定によるカーボンフットプリント最適化 |

### 2-3. AWS 固有の Hint & Tips

- **AWS Direct Connect + Google Cloud Interconnect** の相互接続は、Megaport / Equinix 等のパートナーを経由するか、GCP の **Cross-Cloud Interconnect** を利用する。
- **AWS Transit Gateway** を中心に据え、VPN 経由で GCP VPC と接続する構成も可能（コスト優先時）。
- **AWS IAM** と **Google Cloud Workload Identity Federation** を連携させ、AWS ワークロードから GCP リソースへ長期クレデンシャル不要でアクセスできる（後述）。
- **Amazon S3** と **Google Cloud Storage** 間のデータ同期は AWS DataSync / GCP Storage Transfer Service で実現する。
- **エグレスコスト** — AWS → GCP 方向のデータ転送は AWS 側でエグレス課金が発生する。GCP の "Google-managed private network" 転送や CloudFront 経由でエグレスを最小化できる。
- **Route 53 ヘルスチェック** — GCP 側エンドポイントも監視可能。フェイルオーバールーティングでクロスクラウド DR を実現する。

---

## Step 3 — GCP 側の情報収集とベストプラクティス確認

### 3-1. GCP ベストプラクティス・ドキュメントの取得

`google-developer-knowledge` MCP の `search_documents` / `answer_query` ツールを使い、
対象サービスのベストプラクティスやマルチクラウド関連のガイダンスを取得する。

```
query: "Google Cloud <対象サービス> best practices multicloud with AWS"
```

必要に応じて `get_documents` で該当ドキュメントの全文を取得する。

### 3-2. GCP Well-Architected Framework（Architecture Framework）の観点

GCP の Well-Architected Framework は以下の 6 本柱で構成される。
`google-developer-knowledge` MCP で `query: "Google Cloud Well-Architected Framework <pillar>"` を検索して取得する。

| 柱 (Pillar) | マルチクラウドでの考慮点 |
|---|---|
| 運用の卓越性 (Operational excellence) | Cloud Ops (Monitoring/Logging) と CloudWatch の統合、SLO 定義、自動化 |
| セキュリティ・プライバシー・コンプライアンス | Workload Identity Federation、Cloud KMS と AWS KMS の分離管理 |
| 信頼性 (Reliability) | マルチリージョン展開、クロスクラウド DR、自動復旧 |
| コスト最適化 (Cost optimization) | エグレス削減、Committed Use Discounts (CUD)、Autoclass |
| パフォーマンス最適化 (Performance optimization) | ネットワーク経路最適化、データ配置、リソースのライトサイジング |
| 持続可能性 (Sustainability) | 低炭素リージョンの選定 |

### 3-3. GCP 固有の Hint & Tips

- **Cross-Cloud Interconnect** — Google が Google ネットワークと AWS ネットワークの間に専用物理接続を構築する。10Gbps / 100Gbps 単位、プロビジョニングに 1〜4 週間。冗長化は手動構成。
- **Partner Cross-Cloud Interconnect for AWS** — Cross-Cloud Interconnect の進化版。1Gbps〜100Gbps の柔軟な帯域、**数分でプロビジョニング可能**、冗長性が製品に組込済、GCP / AWS どちらからでも双方向で発注可能。物理インフラ管理・第三者調整が不要。低帯域・ネットワーク専門知識が不要な用途に最適。
- **HA VPN** — Cloud VPN の高可用性構成。AWS VPN Gateway と Site-to-Site VPN (IPsec) で接続する（低コスト・PoC 向け、99.99% SLA）。
- **Network Connectivity Center (NCC)** — ハブ&スポークでオンプレ・複数クラウドを一元管理。Site-to-site data transfer 機能で Google ネットワークを WAN として利用し、AWS ⇔ 他クラウド間の転送も可能。
- **Hybrid NAT** — 両クラウドの IP アドレス空間が重複する場合、接続上で Hybrid NAT を有効化して解決できる。
- **Cloud Storage ⇔ Amazon S3** — Storage Transfer Service で S3 → GCS へ転送可能。エージェントレス転送、イベント駆動転送（S3 Event → SQS）に対応。
- **Workload Identity Federation** — AWS EC2 のインスタンスプロファイル（一時クレデンシャル）を GCP の短期トークンと交換し、サービスアカウントキー無しで GCP API にアクセスできる。
- **エグレスコスト** — GCP → AWS 方向のデータ転送は GCP 側でエグレス課金が発生する。Private Service Connect やキャッシュ戦略で最小化する。

---

## Step 4 — 設計観点の考慮点

### 4-1. ネットワーク接続パターン

| パターン | 構成 | 適用場面 |
|---|---|---|
| **VPN over Internet** | AWS VPN Gateway ↔ GCP HA VPN (Cloud VPN) | PoC・低帯域・コスト優先 |
| **Partner Cross-Cloud Interconnect for AWS** | GCP マネージド接続（1G〜100G、数分で構築） | 本番・柔軟な帯域・迅速構築 |
| **Cross-Cloud Interconnect** | GCP 専用物理接続（10G/100G） | 本番・高帯域・低レイテンシ |
| **パートナー経由** | Direct Connect ↔ Partner Interconnect（Megaport/Equinix） | 柔軟なマルチクラウドファブリック |
| **NCC ハブ&スポーク** | Network Connectivity Center で一元管理 | 複数クラウド・オンプレ統合 |
| **アプリ層連携** | API Gateway + Private Service Connect | マイクロサービス間通信 |

**Hint & Tips:**
- VPN は暗号化オーバーヘッドにより実効帯域が制限される。1Gbps 以上が必要なら Partner Cross-Cloud Interconnect for AWS を検討する（迅速かつ柔軟）。
- 両クラウドのリージョンは地理的に近い組み合わせを選ぶ（例: AWS 東京 `ap-northeast-1` と GCP 東京 `asia-northeast1`）。
- CIDR 設計は両クラウドで重複しないよう事前に統合的に計画する（RFC 1918 範囲を分割）。重複が避けられない場合は GCP の **Hybrid NAT** を利用する。

### 4-2. ID・認証統合

| 要素 | AWS 側 | GCP 側 | 統合方法 |
|---|---|---|---|
| ワークロード認証 | IAM Roles + STS | Workload Identity Federation | AWS 一時クレデンシャルを GCP 短期トークンに交換 |
| 人間の SSO | IAM Identity Center | Cloud Identity / Workforce Identity Federation | SAML 2.0 / OIDC フェデレーション |
| MFA | IAM MFA | Cloud Identity MFA | 両方で有効化 |
| 特権管理 | IAM Roles | IAM + PAM (Privileged Access Manager) | JIT (Just-In-Time) アクセス |

**Workload Identity Federation の 2 方式（AWS → GCP）:**

1. **AWS IAM Credentials 方式（推奨・設定が容易）**
   - AWS の一時セキュリティクレデンシャル（IAM ロール / インスタンスプロファイル）を GCP が **AWS の `GetCallerIdentity` API** で検証する。
   - **AWS 側の設定変更は不要**。GCP 側で Workload Identity Pool と AWS プロバイダーを作成するだけ。
   - 属性マッピング例：
     ```
     google.subject=assertion.arn
     attribute.account=assertion.account
     attribute.aws_role=assertion.arn.extract('assumed-role/{role_name}/')
     attribute.aws_ec2_instance=assertion.arn.extract('assumed-role/{role_and_session}').extract('/{session}')
     ```
   - gcloud での構成例：
     ```
     gcloud iam workload-identity-pools create POOL_ID --location="global"
     gcloud iam workload-identity-pools providers create-aws PROVIDER_ID \
       --location="global" --workload-identity-pool="POOL_ID" \
       --account-id="AWS_ACCOUNT_ID" --attribute-mapping="google.subject=assertion.arn"
     ```

2. **AWS outbound identity federation (OIDC) 方式**
   - AWS が OIDC IdP として動作し、短期 JWT をワークロードに発行する。
   - AWS 側で OIDC issuer URL の設定が必要。

**Hint & Tips:**
- 短期クレデンシャルへの直接リソースアクセス（`principalSet://...`）を推奨。API 制約があるサービスのみサービスアカウントの偽装 (impersonation) を使う。
- IMDSv2 を使う新しい EC2 インスタンスでは、gcloud CLI の手順に従う必要がある。
- 逆方向（GCP → AWS）は AWS IAM の OIDC / SAML ID プロバイダーとして GCP のサービスアカウントを信頼させ、`AssumeRoleWithWebIdentity` を使う。

### 4-3. データ連携パターン

| パターン | 実装例 | 用途 |
|---|---|---|
| **バッチ同期** | Storage Transfer Service (S3→GCS) / AWS DataSync | 定期的なデータレプリケーション |
| **イベント駆動** | S3 Event → SQS → Storage Transfer Service（イベント駆動転送） | リアルタイムデータ連携 |
| **ストリーミング** | Kinesis → Pub/Sub（Kafka プロトコル / コネクタ） | 高スループットデータ連携 |
| **API 連携** | API Gateway (AWS) ↔ API Gateway / Cloud Endpoints (GCP) | サービス間通信 |
| **分析集約** | S3 → BigQuery (BigQuery Omni / Data Transfer Service) | クロスクラウドデータ分析 |

**Storage Transfer Service（S3 → GCS）のエグレス最適化オプション:**

| エグレスオプション | 説明 | コスト特性 |
|---|---|---|
| **Default agentless** | マネージドなエージェントレス転送 | AWS 側でエグレス課金 |
| **CloudFront distribution 経由** | CloudFront をエグレス経路として利用 | S3 直接転送よりエグレスコストが下がる場合あり |
| **Google-managed private network** | Google 管理ネットワーク経由 | **S3 エグレス課金なし**、GCP へ per-GiB 課金（AWS の operation 課金は別途） |
| **Customer-managed private network** | 既存の Cross-Cloud Interconnect / Partner Interconnect / Direct Connect 経由 | プライベート経路、専用線コスト |
| **Agent-driven** | エージェントソフトを導入し経路・帯域を制御 | 全 S3 互換ストレージ対応 |

**Hint & Tips:**
- エグレスコストを最小化するため、データの「出力元」を基準にアーキテクチャを設計する（データの重力）。
- 大量データを AWS → GCP へ移す場合は "Google-managed private network" が S3 エグレスを回避できて効率的。
- 暗号化されたデータの転送時は、双方の KMS / Cloud KMS 鍵管理戦略を事前に整理する。
- イベント駆動転送では、S3 Event Notification を SQS に送り、Storage Transfer Service が SQS を監視して追加・更新分を自動転送する。

### 4-4. DNS 設計

| 要素 | AWS | GCP | 統合方法 |
|---|---|---|---|
| パブリック DNS | Route 53 | Cloud DNS | 委任 or 統合管理 |
| プライベート DNS | Route 53 Private Hosted Zone | Cloud DNS Private Zone | 条件付きフォワーダー |
| クロスクラウド解決 | Route 53 Resolver Outbound | Cloud DNS Server Policy (Forwarding) | フォワーダー相互設定 |

**Hint & Tips:**
- プライベート DNS 解決のために、両クラウドの DNS リゾルバーから相手側フォワーダーへのルートを確保する。
- ドメイン名の命名規則を統一する（例: `*.aws.internal.example.com` / `*.gcp.internal.example.com`）。

---

## Step 5 — 構築観点の考慮点

### 5-1. IaC 戦略

| 観点 | 推奨アプローチ |
|---|---|
| ツール選定 | **Terraform** — 両クラウドを単一ツールで管理可能 |
| State 管理 | AWS: S3 + DynamoDB / GCP: GCS Bucket（object versioning + state locking）— 環境ごとに分離 |
| モジュール構成 | `modules/aws/` + `modules/gcp/` + `modules/shared/` で論理分離 |
| Provider 設定 | 1つの root module で `aws` と `google` 両方を宣言可能 |
| CI/CD | GitHub Actions / GitLab CI / Cloud Build で plan → apply を両クラウド同時に実行 |

**Hint & Tips:**
- 両クラウドの Provider を同一 root module に置く場合、`versions.tf` に双方のバージョンを固定する。
- State ファイルはクラウドごとに分離するのが安全（片方の障害時にも他方の操作を継続可能）。
- GCS バックエンドは object versioning を有効にし、state 破損時に復旧できるようにする。
- `terraform-aws` スキルと `terraform-gcp` スキルを組み合わせて使用する。

### 5-2. 命名規則の統一

```
<org>-<env>-<cloud>-<service>-<role>[-<suffix>]
```

例:
- `myapp-prod-aws-vpc-main`
- `myapp-prod-gcp-vpc-main`
- `myapp-prod-aws-rds-primary`
- `myapp-prod-gcp-sql-primary`

**Hint & Tips:**
- GCP のリソース名には制約（小文字・ハイフン・長さ制限）があるサービスが多い。GCP プロジェクト ID はグローバル一意かつ変更不可なので慎重に設計する。

### 5-3. タグ / ラベル戦略

AWS は「タグ」、GCP は「ラベル」（およびリソース階層のフォルダ / タグ）を使う。両クラウドで統一するキー：

| キー | 説明 | 例 |
|---|---|---|
| `environment` | 環境名 | `production` / `staging` / `development` |
| `project` | プロジェクト名 | `my-app` |
| `owner` | 責任者 | `team-infra` |
| `managed-by` | 管理ツール | `terraform` |
| `cost-center` | コストセンター | `cc-1234` |
| `cloud-provider` | クラウド識別 | `aws` / `gcp` |
| `data-classification` | データ分類 | `confidential` / `internal` / `public` |

**Hint & Tips:**
- **GCP のラベルは小文字英数字・ハイフン・アンダースコアのみ**（キー・値ともに）。AWS のタグは case-sensitive で記号も比較的自由。**両クラウドで使うなら小文字ケバブケース（`cost-center`）に統一する**のが安全。
- GCP はラベル最大 64 個 / リソース、AWS はタグ最大 50 個 / リソース。
- タグ / ラベル未付与リソースの検出には AWS Config Rules + GCP Organization Policy / Security Command Center を併用する。

### 5-4. CI/CD パイプライン設計

```
┌─────────────────────────────────────────────────────────────────┐
│  Source (Git)                                                    │
├───────────────┬───────────────┬─────────────────────────────────┤
│  Lint & Test  │  Lint & Test  │  共通テスト                      │
│  (AWS modules)│ (GCP modules) │  (Cross-cloud integration test)  │
├───────────────┼───────────────┼─────────────────────────────────┤
│  Plan (AWS)   │  Plan (GCP)   │                                 │
├───────────────┼───────────────┤  承認ゲート                      │
│  Apply (AWS)  │  Apply (GCP)  │                                 │
├───────────────┴───────────────┴─────────────────────────────────┤
│  Post-deploy: Cross-cloud connectivity verification              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 6 — 運用観点の考慮点

### 6-1. 監視・可観測性の統合

| 要素 | AWS | GCP | 統合方法 |
|---|---|---|---|
| メトリクス | CloudWatch | Cloud Monitoring | Datadog / Grafana で統合 |
| ログ | CloudWatch Logs | Cloud Logging | SIEM 統合 (Chronicle / SecurityHub) |
| トレース | X-Ray | Cloud Trace | OpenTelemetry で統一 |
| アラート | CloudWatch Alarms | Cloud Monitoring Alerting | PagerDuty / OpsGenie に集約 |
| ダッシュボード | CloudWatch Dashboard | Cloud Monitoring Dashboard | Grafana で統合表示 |

**Hint & Tips:**
- **OpenTelemetry** を計装標準にすれば、バックエンドを自由に切り替えられる。
- **Grafana Cloud** や **Datadog** でマルチクラウドダッシュボードを構築するのが最も効率的。
- GCP の Cloud Monitoring は AWS アカウントを監視対象に追加できる（メトリクススコープ / AWS Connector）。ただし現在の対応状況は最新ドキュメントで確認する。
- インシデント対応の「単一ペイン」を確保するため、アラートは 1 つのインシデント管理ツールに集約する。

### 6-2. コスト管理

| 要素 | AWS | GCP | 統合方法 |
|---|---|---|---|
| コスト可視化 | Cost Explorer | Cloud Billing Reports / BigQuery Export | FinOps ツール (Apptio, Cloudability) |
| 予算アラート | AWS Budgets | Budgets & Alerts | 統一的な閾値管理 |
| コミット割引 | Reserved Instances / Savings Plans | Committed Use Discounts (CUD) | 利用率レポートの統合 |
| エグレス最適化 | - | - | キャッシュ・データ配置・専用線で削減 |

**Hint & Tips:**
- マルチクラウドの最大のコスト要因は **エグレス料金**。データ転送量を可視化・監視する。
- GCP は Cloud Billing データを **BigQuery にエクスポート**でき、AWS の CUR (Cost and Usage Report) と BigQuery で突合してマルチクラウドコスト分析ができる。
- 両クラウドの RI/SP/CUD は個別に最適化するが、FinOps ツールで統合的にカバレッジを管理する。
- ラベル / タグベースのコスト配分を両クラウドで統一し、部門・プロジェクト単位での按分を可能にする。

### 6-3. セキュリティ統制

| 観点 | AWS | GCP | マルチクラウド統合 |
|---|---|---|---|
| ポスチャ管理 | Security Hub | Security Command Center (SCC) | CSPM ツールで統合 |
| コンプライアンス | AWS Config | Organization Policy / SCC | 統一ルールセット |
| 脅威検出 | GuardDuty | Security Command Center / Chronicle | SIEM 統合 |
| 暗号鍵管理 | KMS | Cloud KMS | クラウドごとに分離管理 |
| ネットワークセキュリティ | Security Group / NACL | Firewall Rules / Cloud Armor | 統一ポリシー |
| 監査ログ | CloudTrail | Cloud Audit Logs | 集約ストレージ / SIEM |

**Hint & Tips:**
- **暗号鍵はクラウド境界を超えない**のが原則。AWS KMS の鍵を GCP で使う（逆も）のではなく、各クラウドの鍵管理サービスで個別に管理する。
- **CSPM (Cloud Security Posture Management)** ツール（Prisma Cloud, Wiz, Orca 等）で両クラウドのセキュリティ状態を統一管理する。
- **CloudTrail + Cloud Audit Logs** を1つの SIEM（Google Chronicle or Splunk）に集約し、クロスクラウドの脅威を検出する。
- **ゼロトラスト** — クラウド間通信は常に暗号化し、mTLS やトークン検証を適用する。Workload Identity Federation で長期クレデンシャルの共有を排除する。

### 6-4. DR / BCP 設計

| パターン | 説明 | RPO/RTO |
|---|---|---|
| **Active-Active** | 両クラウドで同時に稼働、Route 53 / Cloud DNS で分散 | RPO≈0 / RTO≈0 |
| **Active-Passive** | 片方をスタンバイ、障害時にフェイルオーバー | RPO: 分〜時間 / RTO: 分〜時間 |
| **Pilot Light** | 最小構成のみ待機、障害時にスケールアップ | RPO: 時間 / RTO: 時間 |
| **Backup & Restore** | データバックアップのみ他方に保管（S3 ⇔ GCS） | RPO: 日 / RTO: 日 |

---

## Step 7 — 構成図（アーキテクチャ図）の作成

照会者が構成図を必要とする場合、`drawio` MCP サーバーを使用して作成する。

### 7-1. AWS 側の構成図

`drawio` の `search_shapes` ツールで AWS アーキテクチャアイコンを検索し、
`open_drawio_xml` ツールで構成図を作成する。

**AWS アーキテクチャアイコン参照:**
https://aws.amazon.com/jp/architecture/icons/

AWS アイコンは `shape=mxgraph.aws4.*` のネイティブベクターシェイプとして実装されている。
使用する主要アイコンの検索例：
```
query: "aws vpc"
query: "aws ec2 instance"
query: "aws rds"
query: "aws lambda"
query: "aws s3"
query: "aws direct connect"
query: "aws transit gateway"
query: "aws route 53"
```

**リージョン / AZ / VPC / サブネットは「公式グループコンテナ」を使う（重要）:**

AWS のリージョン・アベイラビリティゾーン・VPC・サブネットは、単なる角丸四角形ではなく、
**左上に公式アイコンが付いた `mxgraph.aws4.group` コンテナ**として描くのが正式。
`search_shapes` で `query: "aws region group"` / `"aws availability zone group"` /
`"aws vpc subnet private group"` を検索し、`grIcon=` 付きのシェイプを使う。

| 論理階層 | シェイプ（grIcon） | 推奨カラー |
|---|---|---|
| AWS Cloud | `grIcon=mxgraph.aws4.group_aws_cloud_alt` | `#232F3E` |
| Region | `grIcon=mxgraph.aws4.group_region`（点線） | `#00A4A6` |
| Availability Zone | `grIcon=mxgraph.aws4.group_availability_zone`（点線） | `#00A4A6` |
| VPC | `grIcon=mxgraph.aws4.group_vpc2` | `#8C4FFF` |
| Private Subnet | `grIcon=mxgraph.aws4.group_security_group`（塗り） | `#00A4A6` / fill `#E6F6F7` |

- これらのコンテナは `container=1;collapsible=0;recursiveResize=0` を付け、
  **子要素（下位コンテナや EC2/RDS アイコン）を `parent` にネストして親子構造**にすると、
  「Region ＞ VPC ＞ AZ ＞ Subnet ＞ リソース」の階層が正しく表現される。
- 子の座標は親からの相対座標になる点に注意する。

### 7-2. GCP 側の構成図

`drawio` の `search_shapes` ツールで GCP プロダクトアイコンを検索し、
`open_drawio_xml` ツールで構成図を作成する。

**GCP プロダクトアイコン参照:**
https://cloud.google.com/icons?hl=ja#google-cloud-product-icons

GCP アイコンは一部が `shape=mxgraph.gcp2.*` のネイティブベクターシェイプとして実装されている
（例: `mxgraph.gcp2.compute_engine_2`）。
使用する主要アイコンの検索例：
```
query: "gcp compute engine"
query: "gcp vpc"
query: "gcp cloud sql"
query: "gcp cloud storage"
query: "gcp cloud interconnect"
query: "gcp cloud vpn"
query: "gcp cloud dns"
query: "gcp bigquery"
query: "gcp gke kubernetes engine"
```

**GCP アイコンには 2 系統があり、プロダクトによって形式が異なる（重要）:**

1. **ネイティブベクター系** — `shape=mxgraph.gcp2.<name>`。
   `mxgraph.gcp2.compute_engine_2` など。塗り色は `fillColor` で制御。
2. **公式カラーのインライン SVG 系** — `shape=image;...image=data:image/svg+xml,...`。
   **Cloud Load Balancing・Cloud SQL などは、公式アイコンがこのインライン SVG 形式**で提供され、
   `mxgraph.gcp2.*` のネイティブシェイプが存在しない（または簡易版しかない）。
   `search_shapes` の結果で `title` が正しいプロダクト名になっている **インライン SVG のエントリを
   そのまま採用する**（`data:image/svg+xml,...` を含む `style` をコピーして使う）。

   検索例：
   ```
   query: "gcp cloud load balancing icon"   → 公式 Cloud Load Balancing（インライン SVG）
   query: "gcp cloud sql icon"              → 公式 Cloud SQL（インライン SVG）
   ```

**リージョン / ゾーン / サブネットのコンテナの描き方:**

- GCP には **AWS の「Availability Zone グループ」に相当する公式グループアイコンが存在しない**。
  `search_shapes` で GCP のゾーングループを探しても、ヒットするのは AWS の
  `group_availability_zone` なので流用しない。
- そのため GCP 側は、リージョン枠・ゾーン枠を **公式カラー（`#4285F4` 系）の点線角丸コンテナ**で表現する。
- **サブネットはリージョン単位**（ゾーンをまたぐ）なので、
  「サブネット枠の中に zone-a / zone-c の点線枠を内包し、その中に VM を置く」構造で描くと、
  「サブネットは 1 個・VM は複数ゾーンに分散（＝可用性はサブネット分割ではなくゾーン分散で確保）」
  という GCP の正しいモデルが図から読み取れる。AWS 側の「AZ ごとにサブネットを分ける」構造とは
  対比的に描き分けること。

### 7-3. マルチクラウド統合構成図

AWS 側と GCP 側を 1 枚の図にまとめ、相互接続部分を明示する。
以下のレイアウトを基本とする：

```
┌──────────────────────┐         ┌──────────────────────┐
│      AWS Cloud       │◄───────►│      GCP (Google)    │
│  ┌───────────────┐   │  VPN /  │   ┌───────────────┐  │
│  │   VPC         │   │  Cross- │   │    VPC        │  │
│  │  ┌─────────┐  │   │  Cloud  │   │  ┌─────────┐  │  │
│  │  │ Workload│  │   │ Inter-  │   │  │ Workload│  │  │
│  │  └─────────┘  │   │ connect │   │  └─────────┘  │  │
│  └───────────────┘   │         │   └───────────────┘  │
└──────────────────────┘         └──────────────────────┘
```

構成図は **draw.io XML 形式**で作成する。

### 7-4. PNG ファイルへの保存

最終的に構成図を **`draw.png` フォーマット**で `aws-gcp-architecture.draw.png` として
PC 上のフォルダに保存する。

**保存手順:**
1. `open_drawio_xml` で AWS + GCP のマルチクラウド構成図を作成する。
2. `routing: "libavoid"` を指定して、コンテナをまたぐエッジがきれいに配線されるようにする。
3. 図を PNG として保存する（下記のいずれか）。

**PNG 保存の実際 — 専用の保存ツールは無い（重要）:**

- `drawio` MCP には **図をワークスペースに直接 PNG 出力する専用ツールは無い**
  （`open_drawio_xml` はブラウザの app.diagrams.net でエディタを開くだけ）。
  `save_drawio_png` のようなツール名を推測で呼ばないこと（存在しない）。
- したがって PNG 保存は、以下のいずれかをユーザーに案内する運用になる。

  1. **ブラウザからエクスポート（最も手軽）**
     開いた draw.io エディタで「File > Export as > PNG」→
     `aws-gcp-architecture.draw.png` として保存。
     推奨設定: Border 10〜20 / Light 背景 / Zoom 100%。
  2. **Kiro（VS Code）内で完結させる場合**
     Draw.io Integration 拡張（`hediet.vscode-drawio`）を入れ、
     ワークスペース直下に `aws-gcp-architecture.drawio.png`（編集可能な PNG）を作成 →
     図の XML を貼り付けて `Ctrl+S`。ワークスペース内に PNG として直接保存でき、以降も編集可能。
  3. **ソースだけワークスペースに残す**
     `fs_write` で draw.io XML を `aws-gcp-architecture.drawio` として保存しておけば、
     上記拡張で開いて随時 PNG 化できる。エージェントが直接作成できるのはこの XML ソースまで。

**Hint & Tips（AWS × GCP は Azure と違い PNG 崩れが起きにくい）:**
- AWS アイコン（`mxgraph.aws4.*`）も GCP アイコン（`mxgraph.gcp2.*`）も、**draw.io にネイティブ組み込みのベクターシェイプ**として実装されている。
- そのため、Azure シェイプ（外部 SVG 参照 + グラデーション）で起きていた **PNG エクスポート時の画像崩れ問題は AWS × GCP 構成図では基本的に発生しない**。
- ただし GCP アイコンには一部、`shape=image;...image=data:image/svg+xml,...` 形式（インライン SVG）のものも検索結果に混在する。**Cloud Load Balancing・Cloud SQL は公式アイコンがこのインライン SVG 形式**であり、避けられない。インライン SVG は画像データを図に埋め込む形式なので外部参照より崩れにくいが、多用する場合は念のため Web 版 (app.diagrams.net) でのエクスポートを推奨する。
- 迷ったら `mxgraph.gcp2.*` 系のネイティブシェイプを優先的に選ぶことで、エクスポートの安定性が高まる。ネイティブが無いプロダクト（LB・Cloud SQL 等）は公式インライン SVG を使い、独自の色付きボックスで代用しない（公式アイコンでの表現をユーザーが期待するため）。

---

## Step 8 — Terraform コード生成（IaC）

照会者が Terraform でのプロビジョニングを希望する場合、既存スキルを活用する。

### 8-1. AWS 側の Terraform コード

`terraform-aws` スキルを使用し、AWS 環境のコードを生成する。

主な活用場面：
- VPC / サブネット / セキュリティグループ
- EC2 / EKS / Lambda
- RDS / DynamoDB
- Direct Connect / VPN Gateway
- Route 53 / CloudFront
- IAM Roles / Policies / OIDC Provider（GCP → AWS フェデレーション用）

### 8-2. GCP 側の Terraform コード

`terraform-gcp` スキルを使用し、GCP 環境のコードを生成する。

主な活用場面：
- VPC / サブネット / Firewall Rules
- Compute Engine / GKE / Cloud Functions
- Cloud SQL / Firestore / BigQuery
- Cloud Interconnect / HA VPN / Cloud Router
- Cloud DNS / Cloud Load Balancing
- IAM / Workload Identity Federation Pool & Provider

### 8-3. マルチクラウド統合 Terraform 構成

両クラウドを 1 つの Terraform プロジェクトで管理する場合のディレクトリ構成：

```
<project-root>/
├── versions.tf               # AWS + Google 両プロバイダーのバージョン固定
├── variables.tf              # 共通変数（環境名・ラベル/タグ等）
├── outputs.tf                # 統合出力
├── main.tf                   # モジュール呼び出し
│
├── modules/
│   ├── aws/
│   │   ├── networking/       # VPC, Subnet, TGW, VPN
│   │   ├── compute/          # EC2, EKS, Lambda
│   │   ├── database/         # RDS, DynamoDB
│   │   └── security/         # IAM, SG, KMS
│   ├── gcp/
│   │   ├── networking/       # VPC, Subnet, HA VPN, Cloud Router
│   │   ├── compute/          # Compute Engine, GKE, Cloud Functions
│   │   ├── database/         # Cloud SQL, Firestore, BigQuery
│   │   └── security/         # IAM, Firewall, Cloud KMS
│   └── cross-cloud/
│       ├── vpn-connection/       # AWS ↔ GCP VPN 接続
│       ├── dns-forwarding/       # クロスクラウド DNS 解決
│       └── workload-identity/    # Workload Identity Federation
│
└── environments/
    ├── dev/
    ├── stg/
    └── prod/
```

### versions.tf テンプレート（マルチクラウド）

```hcl
##############################################################################
# マルチクラウド Terraform バージョン制約
# AWS プロバイダーと Google プロバイダーを同時に使用する。
##############################################################################
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> <AWS_LATEST_VERSION>"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> <GOOGLE_LATEST_VERSION>"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = var.common_tags
  }
}

provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
  # ラベルは各リソースで var.common_labels をマージして付与する
}
```

> `terraform` MCP の `get_latest_provider_version` で
> `hashicorp/aws` と `hashicorp/google` の最新バージョンを取得して埋める。

---

## Step 9 — マルチクラウド設計のアンチパターン

避けるべきアプローチを明確にする。

| アンチパターン | 問題点 | 推奨代替案 |
|---|---|---|
| 最小公倍数アーキテクチャ | 両方のクラウドで同一設計を強制し、各社の強みを殺す | ベスト・オブ・ブリードで適材適所 |
| クラウド間の密結合 | 障害のカスケードが発生する | 非同期・イベント駆動で疎結合に |
| 中央集権的な鍵管理 | 一方の障害で全体が止まる | クラウドごとに鍵を分離管理 |
| 単一 State ファイル | 片方の変更が他方に影響するリスク | クラウドごとに State を分離 |
| エグレスを意識しない設計 | コストが爆発する | データの重力 (Data Gravity) を考慮 |
| 長期クレデンシャルの共有 | 漏洩リスク・ローテーション負荷 | Workload Identity Federation で短期トークン化 |
| 監視の二重管理 | 統合ビューがなく障害発見が遅れる | 統合ダッシュボード＋単一アラート先 |

---

## Step 10 — 最終レビューチェックリスト

マルチクラウド設計・構築のレビュー時に確認する。

### 設計

- [ ] CIDR が両クラウドで重複していないか（重複時は Hybrid NAT を検討したか）
- [ ] ネットワーク接続パターンが要件（帯域・レイテンシ・コスト）に合致しているか
- [ ] ID 統合が設計されており、Workload Identity Federation で長期クレデンシャルの共有がないか
- [ ] データ連携パターンがエグレスコストを考慮しているか（Storage Transfer Service の経路選択）
- [ ] DNS 設計でクロスクラウドの名前解決が可能か
- [ ] DR / BCP の RPO / RTO が要件を満たしているか

### 構築

- [ ] Terraform の State がクラウドごとに分離されているか（GCS は versioning 有効か）
- [ ] 命名規則・ラベル/タグ付けが両クラウドで統一されているか（小文字ケバブケース）
- [ ] CI/CD パイプラインが両クラウドのデプロイを管理しているか
- [ ] 暗号化がクラウドごとの鍵管理サービス（KMS / Cloud KMS）で実装されているか
- [ ] クロスクラウド接続リソース（VPN / Interconnect）が冗長化されているか

### 運用

- [ ] 統合監視ダッシュボードが構築されているか
- [ ] アラートが単一のインシデント管理ツールに集約されているか
- [ ] コスト可視化が両クラウドを横断しているか（BigQuery でコスト突合）
- [ ] セキュリティポスチャ管理が両クラウドをカバーしているか（Security Hub + SCC）
- [ ] CloudTrail + Cloud Audit Logs が SIEM に集約されているか
- [ ] インシデント対応手順にクロスクラウドシナリオが含まれているか

---

## 付録 A — サービス対応表

両クラウドの主要サービスの対応関係を示す。

| カテゴリ | AWS | GCP (Google Cloud) |
|---|---|---|
| コンピュート (VM) | EC2 | Compute Engine |
| コンテナ (マネージド K8s) | EKS | GKE (Google Kubernetes Engine) |
| サーバーレス | Lambda | Cloud Functions / Cloud Run |
| オブジェクトストレージ | S3 | Cloud Storage |
| RDBMS (マネージド) | RDS / Aurora | Cloud SQL / AlloyDB / Spanner |
| NoSQL | DynamoDB | Firestore / Bigtable |
| データウェアハウス | Redshift | BigQuery |
| メッセージキュー | SQS | Pub/Sub |
| イベントバス | EventBridge | Eventarc |
| CDN | CloudFront | Cloud CDN |
| DNS | Route 53 | Cloud DNS |
| VPN | Site-to-Site VPN | Cloud VPN (HA VPN) |
| 専用線 | Direct Connect | Cloud Interconnect / Cross-Cloud Interconnect |
| ネットワークハブ | Transit Gateway | Network Connectivity Center (NCC) |
| ID 管理 | IAM / IAM Identity Center | Cloud IAM / Cloud Identity |
| ワークロード認証連携 | IAM Roles + OIDC/SAML | Workload Identity Federation |
| 鍵管理 | KMS | Cloud KMS |
| シークレット管理 | Secrets Manager | Secret Manager |
| 監視 | CloudWatch | Cloud Monitoring |
| ログ分析 | CloudWatch Logs Insights | Cloud Logging (Log Analytics) |
| 監査ログ | CloudTrail | Cloud Audit Logs |
| コンプライアンス / ポスチャ | Config / Security Hub | Organization Policy / Security Command Center |
| IaC (ネイティブ) | CloudFormation | Cloud Deployment Manager / Config Controller |
| コンテナレジストリ | ECR | Artifact Registry |
| API ゲートウェイ | API Gateway | API Gateway / Cloud Endpoints |
| WAF | AWS WAF | Cloud Armor |
| データ転送 | DataSync | Storage Transfer Service |

---

## 付録 B — クロスクラウド接続の Terraform 実装例

### B-1. HA VPN 接続（AWS VPN Gateway ↔ GCP HA VPN）

AWS Site-to-Site VPN ↔ GCP HA VPN の接続の実装パターン（要点）：

```hcl
##############################################################################
# Cross-Cloud VPN: AWS ↔ GCP
# AWS 側: VPN Gateway + Customer Gateway
# GCP 側: HA VPN Gateway + Cloud Router + External VPN Gateway
##############################################################################

# --- GCP 側 ---
resource "google_compute_ha_vpn_gateway" "main" {
  name    = "${var.prefix}-ha-vpn-gw"
  region  = var.gcp_region
  network = google_compute_network.main.id
}

resource "google_compute_router" "main" {
  name    = "${var.prefix}-cloud-router"
  region  = var.gcp_region
  network = google_compute_network.main.id

  bgp {
    asn = 65001
  }
}

resource "google_compute_external_vpn_gateway" "aws" {
  name            = "${var.prefix}-aws-ext-gw"
  redundancy_type = "TWO_IPS_REDUNDANCY"

  interface {
    id         = 0
    ip_address = aws_vpn_connection.to_gcp.tunnel1_address
  }
  interface {
    id         = 1
    ip_address = aws_vpn_connection.to_gcp.tunnel2_address
  }
}

# --- AWS 側 ---
resource "aws_vpn_gateway" "main" {
  vpc_id = module.aws_networking.vpc_id
  tags   = merge(var.common_tags, { Name = "${var.prefix}-vpn-gw" })
}

resource "aws_customer_gateway" "gcp" {
  bgp_asn    = 65001 # GCP Cloud Router の ASN
  ip_address = google_compute_ha_vpn_gateway.main.vpn_interfaces[0].ip_address
  type       = "ipsec.1"
  tags       = merge(var.common_tags, { Name = "${var.prefix}-gcp-cgw" })
}

resource "aws_vpn_connection" "to_gcp" {
  vpn_gateway_id      = aws_vpn_gateway.main.id
  customer_gateway_id = aws_customer_gateway.gcp.id
  type                = "ipsec.1"
  tags                = merge(var.common_tags, { Name = "${var.prefix}-vpn-gcp" })
}
```

> **注意:** Pre-Shared Key やトンネル設定は `sensitive = true` 変数で管理するか、
> Secrets Manager / Secret Manager から取得することを推奨する。
> トンネル 2 本（HA 構成）と BGP セッションの設定が別途必要。

### B-2. Workload Identity Federation（AWS → GCP、キー不要）

```hcl
##############################################################################
# Workload Identity Federation: AWS EC2 → GCP
# AWS 側の設定変更は不要（AWS IAM Credentials 方式）
##############################################################################

resource "google_iam_workload_identity_pool" "aws_pool" {
  workload_identity_pool_id = "${var.prefix}-aws-pool"
  display_name              = "AWS Workload Pool"
  description               = "Trust AWS workloads to access GCP without keys"
}

resource "google_iam_workload_identity_pool_provider" "aws_provider" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.aws_pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "${var.prefix}-aws-provider"
  display_name                       = "AWS Provider"

  aws {
    account_id = var.aws_account_id
  }

  attribute_mapping = {
    "google.subject"        = "assertion.arn"
    "attribute.account"     = "assertion.account"
    "attribute.aws_role"    = "assertion.arn.extract('assumed-role/{role_name}/')"
  }
}

# 特定の AWS ロールに GCP リソースへのアクセスを付与する例
resource "google_storage_bucket_iam_member" "aws_role_access" {
  bucket = google_storage_bucket.shared.name
  role   = "roles/storage.objectViewer"
  member = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.aws_pool.name}/attribute.aws_role/${var.aws_role_name}"
}
```

---

## 付録 C — 利用する MCP ツール一覧

| 用途 | MCP サーバー | 主なツール |
|---|---|---|
| AWS ドキュメント検索 | aws-mcp | `search_documentation`, `read_documentation` |
| AWS リソース操作 | aws-mcp | `call_aws`, `run_script` |
| GCP ドキュメント検索 | google-developer-knowledge | `search_documents`, `answer_query` |
| GCP ドキュメント全文取得 | google-developer-knowledge | `get_documents` |
| 構成図作成 | drawio | `search_shapes`, `open_drawio_xml` |
| Terraform バージョン確認 | terraform | `get_latest_provider_version` |
| Terraform モジュール検索 | terraform | `search_modules`, `get_module_details` |
| AWS Terraform コード | terraform-aws スキル | (スキル連携) |
| GCP Terraform コード | terraform-gcp スキル | (スキル連携) |

---

## 付録 D — アイコン参照 URL

| クラウド | アイコン参照 URL | draw.io シェイプ接頭辞 |
|---|---|---|
| AWS | https://aws.amazon.com/jp/architecture/icons/ | `shape=mxgraph.aws4.*`（ネイティブ） |
| GCP | https://cloud.google.com/icons?hl=ja#google-cloud-product-icons | `shape=mxgraph.gcp2.*`（ネイティブ） |
