# 引き継ぎメモ: homelab-cluster リポジトリ作成

このファイルは、`arumono-recipe` の検討セッションから派生した `homelab-cluster` リポジトリ作成タスクを、別セッションへ引き継ぐためのコンテキストである。

## 1. これまでの経緯

- `arumono-recipe` は「家にある食材から献立を提案するPWA」の新規プロジェクト
- 要件定義・技術スタック選定・リポジトリ戦略の3ドキュメントを `arumono-recipe/docs/` に作成済み
- リポジトリ戦略として **分離型（基盤リポ + サービスごとリポ）** を採用
- インフラはOCI Always Free上にk8s (kubeadm) を構築し、複数サービスを相乗りさせる方針
- 既存のk8s環境は削除し、`homelab-cluster` リポで一から再構築する判断
- 順序として、**インフラ完成 → arumono-recipe デプロイ** の流れで進める

## 2. 既決事項（要点）

### 2.1 インフラ前提

- ホスト: OCI Always Free / Ampere A1 (ARM, 4 OCPU / 24GB RAM) × 1ノード
- Kubernetes: **kubeadm** シングルノード（CNI は Flannel または Calico を別途導入）
- Ingress: `ingress-nginx`（サブドメイン振り分け）
- TLS: `cert-manager` + Let's Encrypt (HTTP-01)
- ストレージ: OCI Block Volume Always Free 200GB
- ロードバランサ: OCI Always Free LB (10Mbps) または Public IP 直結

### 2.2 共通基盤コンポーネント

- ingress-nginx
- cert-manager
- sealed-secrets
- ArgoCD（GitOps pull型）
- kube-prometheus-stack（Prometheus + Grafana + node-exporter）
- Loki（ログ収集）
- CloudNativePG（PostgreSQL オペレータ。各サービスの DB を Namespace ごとに提供）
- MinIO operator（オブジェクトストレージ。各サービスが必要に応じ利用）

### 2.3 GitOps とリポジトリ責務

- `homelab-cluster/apps/{service}.yaml` に ArgoCD Application を定義
- 各サービスリポの `deploy/overlays/prod` を参照
- サービス側はマニフェスト整備、共通基盤側は Application 登録のみ

### 2.4 命名規約

- Namespace = リポジトリ名 = サービス名（小文字ケバブケース）
- ホスト名: `{サービス名}.{ベースドメイン}` （ベースドメインは未確定）
- イメージ: `ghcr.io/{account}/{サービス名}:{git-sha}`

### 2.5 IaC・CI/CD

- Terraform (OCI provider): VM・VCN・Block Volume・Public IP
- Helmfile または ArgoCD App-of-Apps: 共通基盤の宣言的管理
- GitHub Actions: PR時 lint/test、main マージ時 terraform plan/apply（手動承認）

## 3. 参照すべき既存ドキュメント

`~/git/arumono-recipe/docs/` 配下:

- `requirements.md` ... arumono-recipe の要件定義書（参考程度）
- `tech-stack.md` ... 技術スタック選定書（インフラ部分が直接関係）
- `repository-strategy.md` ... リポジトリ戦略書（homelab-cluster の責務がここに定義されている）

## 4. 新セッションのスコープ

`~/git/homelab-cluster` を新規ディレクトリとして作成し、以下を整える。

### 4.1 Must（初期セットアップ）

1. リポジトリ初期化（`git init` + README.md + .gitignore）
2. ディレクトリ構造の骨格作成（terraform / bootstrap / platform / apps / docs）
3. インフラ設計書 `docs/infrastructure.md` 作成
   - 構成図、ネットワーク図、Namespace 設計、TLS・DNS方針、バックアップ方針
4. Terraform スケルトン（OCI VM / VCN / Subnet / Security List / Public IP / Block Volume）
5. kubeadm ブートストラップスクリプト（kubeadm init / CNI 導入 / kubeconfig 取得手順）

### 4.2 Should（基盤導入）

1. ingress-nginx の Helm values
2. cert-manager の Helm values + ClusterIssuer (staging / prod)
3. sealed-secrets のセットアップ手順
4. ArgoCD インストールと初期 App-of-Apps 構成
5. CloudNativePG オペレータの導入

### 4.3 Could（拡張）

1. kube-prometheus-stack + Loki の導入
2. MinIO operator の導入
3. ArgoCD Application 例として `apps/arumono-recipe.yaml` のサンプルを置く（実体は arumono-recipe 側の `deploy/` が出来てから有効化）

### 4.4 残課題（事前に決める）

- ベースドメイン（`example.com` 部分）
- OCI のリージョン（Ampere A1 在庫状況）
- ArgoCD の認証方式
- GitHub アカウント名（イメージ名・リポジトリURL確定のため）

## 5. 既存環境の前提

- ユーザーは既存のk8sを **削除して再構築する** 方針
- OCI に Ampere A1 のインスタンスを既に保有している
- SSH で OCI インスタンスに接続できる前提

## 6. 進め方の推奨

1. まず `docs/infrastructure.md` でアーキテクチャを固める（IPA風の章立て）
2. Terraform の `terraform/` を書き、ローカルから `terraform plan` できる状態にする
3. `bootstrap/` に kubeadm init スクリプトと CNI 導入手順を置く
4. `platform/` を Helmfile or ArgoCD App-of-Apps いずれかで構築（先に決める）
5. ArgoCD まで稼働したら、arumono-recipe 側に戻り `deploy/` を整備して連携

## 7. 注意点

- 機密情報（OCI APIキー、kubeconfig、サービスアカウントキー）は Git にコミットしない
- Terraform state は当初ローカル管理でも可だが、最終的に OCI Object Storage に backend を切る
- Helm chart / values は バージョン固定（latest禁止）
- 既存環境削除前にバックアップが必要なものがないか確認する
