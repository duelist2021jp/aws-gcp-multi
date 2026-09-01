# AWS x GCP Multi-Cloud Skill

AWS と Google Cloud を併用するマルチクラウド環境の、設計・構築・運用を支援する Kiro スキルです。
ネットワーク接続、ID 統合、データ連携、Terraform、可観測性、コスト管理、セキュリティ統制、DR/BCP を横断して、要件に応じた実装方針を整理します。

詳細な手順と設計ガイドラインは [SKILL.md](SKILL.md) を参照してください。

## 対象となるケース

- AWS と GCP 間を VPN、Cross-Cloud Interconnect、またはパートナー接続で連携したい
- AWS のワークロードから、サービスアカウントキーを持たずに GCP リソースへアクセスしたい
- Amazon S3 と Cloud Storage のデータ転送、または BigQuery への分析集約を設計したい
- Terraform で両クラウドのインフラとクロスクラウド連携を管理したい
- 監視、セキュリティ、コスト、DR を両クラウド横断で統制したい

## 提供するガイダンス

| 領域 | 主な内容 |
| --- | --- |
| 要件整理 | 採用目的、対象ワークロード、接続、ID、データ、運用・セキュリティ要件の確認項目 |
| ネットワーク | VPN、Partner Cross-Cloud Interconnect for AWS、Cross-Cloud Interconnect、NCC、DNS フォワーディング |
| ID とセキュリティ | AWS IAM と Google Cloud IAM の連携、Workload Identity Federation、鍵管理、ゼロトラスト |
| データ連携 | Storage Transfer Service、DataSync、イベント駆動転送、ストリーミング、BigQuery 分析集約 |
| IaC と CI/CD | AWS/Google Provider を用いる Terraform、State 分離、命名規則、タグ/ラベル戦略 |
| 運用 | OpenTelemetry、統合監視、SIEM、FinOps、セキュリティポスチャ管理、DR/BCP |
| 構成図 | draw.io を使用した AWS/GCP 構成図と公式アイコンの利用方法 |

## 利用方法

このディレクトリを Kiro のスキルディレクトリに配置し、AWS と GCP をまたぐ設計・実装・レビューを依頼してください。

依頼例:

```text
AWS 東京リージョンの VPC と GCP 東京リージョンの VPC を冗長 VPN で接続する設計を作成してください。
CIDR、BGP、DNS 解決、監視、Terraform の構成も含めてください。
```

```text
AWS EC2 から Cloud Storage にアクセスするため、Workload Identity Federation をキーなしで構成してください。
最小権限の IAM 設計と Terraform 例を提示してください。
```

```text
S3 のデータを BigQuery に集約する構成を提案してください。
転送経路ごとのエグレスコスト、暗号化、障害時の再実行方法を比較してください。
```

## 前提・連携先

このスキルは、必要に応じて次の MCP サーバーおよび関連スキルを利用します。

| 用途 | 連携先 |
| --- | --- |
| AWS のドキュメント検索・リソース操作 | `aws-mcp` |
| GCP のドキュメント検索 | `google-developer-knowledge` |
| 構成図の作成 | `drawio` |
| Terraform Provider/Module の確認 | `terraform` |
| AWS の Terraform コード生成 | `terraform-aws` スキル |
| GCP の Terraform コード生成 | `terraform-gcp` スキル |

利用可能なツールやスキル名は、実行環境に合わせて読み替えてください。

## 設計上の原則

- クラウドごとの強みを活用し、最小公倍数の設計を避ける
- CIDR、DNS、命名規則、タグ/ラベルを両クラウドで一貫させる
- データの配置と転送量を先に設計し、エグレスコストを管理する
- 長期クレデンシャルを共有せず、短期トークンによるフェデレーションを優先する
- 暗号鍵と Terraform State はクラウド境界ごとに分離する
- 監視、アラート、監査ログを横断的に集約し、DR の RPO/RTO を明文化する

## 収録内容

[SKILL.md](SKILL.md) には、以下を収録しています。

- ネットワーク接続、ID 統合、データ連携、DNS の選定ガイド
- Terraform のディレクトリ構成と AWS/GCP Provider のテンプレート
- HA VPN と Workload Identity Federation の Terraform 実装例
- 運用、コスト、セキュリティ、DR/BCP のレビュー観点
- マルチクラウド設計のアンチパターンと最終チェックリスト
- AWS/GCP のサービス対応表と構成図作成のガイド

## 注意事項

- クラウドサービスの仕様、料金、提供リージョン、MCP ツールの可用性は変更されるため、実装前に各公式ドキュメントで確認してください。
- 本番環境への変更前に、Terraform の実行計画、権限、接続冗長性、バックアップと復旧手順をレビューしてください。
- シークレット、アクセスキー、VPN の事前共有鍵をリポジトリへ保存しないでください。

## ライセンス

このリポジトリのライセンスに従います。