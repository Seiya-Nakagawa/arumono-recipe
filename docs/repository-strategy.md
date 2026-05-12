# リポジトリ戦略

## 1. はじめに

### 1.1 文書の目的

本文書は、arumono-recipe を含む複数サービスを1つのk8sクラスタに相乗りさせる前提で、GitHub上のリポジトリをどう分割し運用するかを定める。

### 1.2 関連文書

- `docs/requirements.md`（要件定義書）
- `docs/tech-stack.md`（技術スタック選定書）

## 2. 採用方針

**分離型（基盤リポ + サービスごとリポ）** を採用する。

### 2.1 採用理由

- **ポートフォリオ最適**: GitHubプロフィールのピン留め枠を使い、複数サービスを一覧で示せる
- **採用担当の動線に合致**: 各サービスのREADMEが独立して完結し、数分の確認で技術スタック・成果物が伝わる
- **責務の独立**: クラスタ運用（基盤）と各サービス開発を別ライフサイクルで扱える
- **段階的拡張**: サービス追加時は新規リポを増やすだけで済む

### 2.2 採用しない選択肢

| 不採用 | 理由 |
| --- | --- |
| モノレポ（全サービス + インフラ同居） | ポートフォリオでの一覧性が弱い。Turborepo等の運用そのものをアピールしたい場合は別 |
| 完全ポリレポ（アプリとデプロイマニフェストも別リポ） | 個人開発では過剰。デプロイPRが分散して回しづらい |

## 3. リポジトリ構成

```text
github.com/{account}/
├── homelab-cluster        ← クラスタ管理・共通基盤
├── arumono-recipe         ← 本サービス（アプリ + マニフェスト）
└── {future-service}       ← 将来追加サービス
```

### 3.1 homelab-cluster

```text
homelab-cluster/
├── README.md              アーキテクチャ図・構築手順・運用ノート
├── terraform/             OCI VM・VCN・Block Volume・Public IP
├── bootstrap/             kubeadm init / kubeconfig 取得スクリプト
├── platform/              共通基盤 (Helmfile or ArgoCD App-of-Apps)
│   ├── ingress-nginx/
│   ├── cert-manager/
│   ├── sealed-secrets/
│   ├── argocd/
│   ├── kube-prometheus-stack/
│   ├── loki/
│   └── cloudnative-pg/    PostgreSQL オペレータ
├── apps/                  ArgoCD Application 定義（各サービスを参照）
│   ├── arumono-recipe.yaml
│   └── {future-service}.yaml
└── docs/                  運用手順・障害対応メモ
```

責務:

- OCI 上のインフラ構築（Terraform）
- k8s クラスタのライフサイクル管理（kubeadm upgrade / certs renew 等）
- クラスタ共通基盤の導入と更新
- GitOps（ArgoCD）の中枢
- クラスタワイドな監視・ログ・シークレット管理
- Namespace / RBAC / ResourceQuota の発行

### 3.2 arumono-recipe（本リポジトリ）

```text
arumono-recipe/
├── README.md
├── docs/                  要件定義・技術選定・基本設計
├── app/                   Next.js アプリ
├── prisma/                スキーマ・マイグレーション
├── Dockerfile
├── compose.yaml           ローカル開発用 (Postgres / MinIO)
├── .github/workflows/     CI/CD
└── deploy/                k8s マニフェスト
    ├── base/              Kustomize base
    └── overlays/
        ├── dev/
        └── prod/
```

責務:

- アプリケーション実装
- アプリのDockerイメージビルドと ghcr.io への push
- 自サービスの k8s マニフェスト（Deployment / Service / Ingress / PVC / ConfigMap）
- 自サービスのスキーマ管理（Prisma）
- 単体テスト・E2E テスト

### 3.3 将来追加サービス

`arumono-recipe` と同じ構成（`app/` + `deploy/`）を踏襲する。命名は **小文字ケバブケース** で統一する。

## 4. 責務分界点

| 項目 | homelab-cluster | サービスリポ |
| --- | --- | --- |
| インフラ（OCI / Terraform） | ◎ | ✕ |
| k8s 本体 (kubeadm) | ◎ | ✕ |
| Ingress Controller / cert-manager | ◎ | ✕ |
| ArgoCD 設定 | ◎ | ✕（Application 定義のみ） |
| 監視スタック | ◎ | ✕ |
| Namespace の作成 | ◎ | ✕ |
| サービス用 Ingress リソース | ✕ | ◎ |
| Deployment / Service / PVC | ✕ | ◎ |
| アプリの Secret 内容 (sealed-secrets化) | ✕ | ◎ |
| ServiceMonitor / PrometheusRule | ✕ | ◎ |

## 5. 命名・ホスト名規約

| 対象 | 規約 | 例 |
| --- | --- | --- |
| リポジトリ名 | サービス名そのまま、小文字ケバブケース | `arumono-recipe` |
| Namespace 名 | リポジトリ名と同一 | `arumono-recipe` |
| ホスト名 | `{サービス名}.{ベースドメイン}` | `arumono.example.com` |
| Docker イメージ | `ghcr.io/{account}/{サービス名}:{git-sha}` | `ghcr.io/your-account/arumono-recipe:abc1234` |
| ArgoCD Application 名 | リポジトリ名と同一 | `arumono-recipe` |

## 6. README 規約（各サービスリポ共通）

採用担当者が数分で価値を把握できるよう、最低限以下を盛り込む。

1. 1行サマリ（何を解決するか）
2. スクリーンショット または デモURL
3. 技術スタックバッジ（Next.js / TypeScript / Vertex AI 等）
4. アーキテクチャ図（簡易テキスト図でも可）
5. ローカル起動手順（`pnpm install && pnpm dev` レベルで動く）
6. デプロイフロー（CI/CD 図）
7. ディレクトリ構成説明
8. ライセンス・作者リンク

## 7. CI/CD の連携

| トリガー | 場所 | 動作 |
| --- | --- | --- |
| PR 作成 | サービスリポ | lint / type-check / unit test |
| main マージ | サービスリポ | イメージビルド → ghcr.io push → `deploy/overlays/prod` の image tag を更新する PR を自動作成（または同コミットに含める） |
| `deploy/` 配下の差分検知 | homelab-cluster の ArgoCD | サービスリポを pull して同期 |
| 共通基盤の更新 | homelab-cluster | Helmfile diff / ArgoCD App-of-Apps |

## 8. 段階的移行プラン

1. **Phase 0**: 既存k8sを削除し、ローカルから OCI に SSH できる状態を確認
2. **Phase 1**: `homelab-cluster` リポジトリを新規作成し、Terraform で OCI リソースを構築・kubeadm で k8s をブートストラップ
3. **Phase 2**: `homelab-cluster/platform/` に nginx-ingress + cert-manager + sealed-secrets + ArgoCD を導入
4. **Phase 3**: `arumono-recipe` の `deploy/` を整備し、ArgoCD Application として `homelab-cluster/apps/arumono-recipe.yaml` を追加
5. **Phase 4**: 監視スタック (kube-prometheus-stack + Loki) を `platform/` に追加
6. **Phase 5**: 2サービス目を追加し、Namespace 分離・ResourceQuota の運用検証

## 9. リスクと緩和策

| リスク | 緩和策 |
| --- | --- |
| homelab-cluster とサービスリポのバージョンずれ | ArgoCD Application で revision を固定。サービスリポのタグ運用を徹底 |
| Secret の取り扱いミス | sealed-secrets で暗号化したものだけを Git にコミット。鍵は homelab-cluster 内で管理 |
| 1リポ内に複数サービスを増やしたくなる衝動 | 「1リポ = 1サービス = 1 Namespace」原則を README に明記 |
| GitHub プライベートリポでポートフォリオ性が落ちる | 公開可能なものは Public、機密情報は別途 Private に切り分け |

## 10. 残課題

- ベースドメインの確定
- ArgoCD の認証方式（OIDC連携 or 単純パスワード）
- GitHub Actions から OCI/k8s へのデプロイ経路の最終形（pull型 ArgoCD のみで完結させるか）
- homelab-cluster を Public にする場合の機密情報の追加レビュー
