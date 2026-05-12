# arumono-recipe 技術スタック選定書

## 1. はじめに

### 1.1 文書の目的

本文書は、`docs/requirements.md` で定義した要件を実現するために採用する技術スタックを選定し、構成と理由を記録することを目的とする。詳細な物理設計・配線は後続の基本設計書で扱う。

### 1.2 関連文書

- `docs/requirements.md`（要件定義書）
- `docs/repository-strategy.md`（リポジトリ戦略）

## 2. 選定方針

要件定義書から導出した重要要件と、選定への影響を以下に示す。

| 要件 | 概要 | 選定への影響 |
| --- | --- | --- |
| スマホ中心のPWA | 片手操作・ホーム追加 | 主流フレームワーク + PWA対応ライブラリを採用 |
| 世帯単位アクセス制御 | 家族グループ間でデータ分離 | RLS や ORM レベルで分離しやすい構成 |
| AIレシピ提案 / 画像生成 / OCR | LLM + Vision + 画像生成 | Google Cloud (Vertex AI / Document AI) で一本化 |
| 個人利用のコスト抑制 | 月額を最小化したい | インフラはOCI Always Freeで無料、AIのみ従量課金 |
| 学習目的でk8s・nginxを使う | 運用学習を兼ねたい | kubeadm + nginx-ingress + cert-manager の自前構成 |
| ハイブリッド方式 | DB蓄積 + AI補完 | 自前PostgreSQLでレシピを永続化 |
| レシート画像の短期保管 | OCR後に削除 | オブジェクトストレージ層を別途設ける |

## 3. 推奨スタック一覧

| レイヤー | 採用技術 | 主な役割 |
| --- | --- | --- |
| フロントエンド | Next.js 15 (App Router) + TypeScript | UI・PWA・ルーティング |
| UI | Tailwind CSS + shadcn/ui | スマホ最適化されたコンポーネント |
| PWA | Serwist (旧 next-pwa) | Service Worker / オフラインキャッシュ |
| 状態管理 | TanStack Query + Zustand | サーバ状態と軽量クライアント状態 |
| バックエンド | Next.js Route Handlers + Server Actions | API・サーバサイドロジック |
| 認証 | Auth.js (NextAuth) + Google OAuth | ログインと世帯メンバー識別 |
| ORM | Prisma | スキーマ管理・型安全クエリ |
| データベース | PostgreSQL 16 (CloudNativePG on k8s) | 主データストア |
| オブジェクトストレージ | MinIO (S3互換, on k8s) | レシート画像の短期保管 |
| LLM | Vertex AI Gemini 2.5 Flash / Pro | レシピ生成・レシート補正 |
| 画像生成 | Vertex AI Imagen 3 | 完成イメージ生成 |
| OCR | Google Document AI (Expense Parser) | レシート文字認識 |
| Ingress / TLS | nginx-ingress-controller + cert-manager + Let's Encrypt | HTTPS終端・ルーティング |
| Kubernetes | kubeadm (シングルノード) | コンテナオーケストレーション（学習目的） |
| インフラ | OCI Always Free (Ampere A1, 4 OCPU / 24GB) | 計算リソース |
| コンテナレジストリ | GitHub Container Registry (ghcr.io) | イメージ配布 |
| CI/CD | GitHub Actions | ビルド・テスト・デプロイ |
| 監視 | Prometheus + Grafana + Loki | メトリクス・ログ可視化 |
| シークレット管理 | Kubernetes Secrets + sealed-secrets | APIキー等の暗号化保管 |
| IaC | Terraform (OCI) + Helmfile | クラスタ外/内のリソース定義 |

## 4. 各レイヤーの選定理由と代替候補

### 4.1 フロントエンド

- **Next.js 15 + TypeScript** を採用する
- 採用理由
  - App Router + RSC によりサーバサイドでLLM呼び出しを行いやすい
  - Server Actions により薄いAPIで完結する
  - PWA・画像最適化・i18n のエコシステムが充実
  - shadcn/ui との相性が良くスマホUIを短時間で組める
- 代替候補: Nuxt / SvelteKit / Remix（学習コストやエコシステム差で次点）

### 4.2 UI・PWA

- **Tailwind CSS + shadcn/ui** をベースに、PWA化は **Serwist** を使う
- Serwist は next-pwa の後継で App Router 対応
- ホーム追加・キャッシュ・オフライン用フォールバックページを実装する

### 4.3 認証

- **Auth.js (NextAuth) + Google OAuth** を採用する
- 採用理由
  - Google系AIサービスを使うため、Google OAuth と相性が良い
  - メール+パスワード方式より個人運用での負担が小さい（パスワードリセット等の独自実装が不要）
- 世帯管理はアプリ側で `Household` / `HouseholdMember` テーブルを保持する
- 招待コードは UUID v4 ベースで生成し、有効期限を設ける

### 4.4 データベース

- **PostgreSQL 16** を **CloudNativePG オペレータ** で k8s 上にデプロイする
- 採用理由
  - リレーション・JSONB・拡張機能（pgvector）に強く、将来のレシピ類似検索にも対応可
  - CloudNativePG はバックアップ・障害復旧・PITR を備え、学習にも有用
- ORM は **Prisma** を採用し、スキーマ駆動でマイグレーションを管理
- 代替候補: Supabase (BaaS) — 学習目的に反するため不採用

### 4.5 オブジェクトストレージ

- **MinIO** を k8s 上に立て、S3互換APIでレシート画像を扱う
- レシート画像は OCR完了後に自動削除する（バケットライフサイクル設定）
- 代替候補: OCI Object Storage Always Free 枠 (20GB) — 学習比重を下げたい場合に切り替え可

### 4.6 AI / 外部サービス（Google系主体）

- **LLM**: Vertex AI Gemini
  - 通常時: `gemini-2.5-flash`（コスト重視）
  - 提案精度を高めたいケース: `gemini-2.5-pro`
- **画像生成**: Vertex AI Imagen 3
  - レシピ完成画像の生成。生成済み画像は MinIO に保存し、再生成を抑制する
- **OCR**: Google Document AI
  - 既製の **Expense Parser**（レシート特化）を使い、商品名・数量・単価を構造化抽出
  - 不安定箇所は Gemini で後段補正（ハイブリッド）
- アダプタ層を `lib/ai/` に切り、プロバイダ差し替え可能な設計とする
- 認証はサービスアカウントキーを Kubernetes Secret として注入する

### 4.7 インフラ（OCI Always Free + k8s (kubeadm) + nginx, 複数サービス相乗り前提）

- **方針**: 既存k8s環境は削除し、`homelab-cluster` リポジトリで一から再構築する。1クラスタを共通基盤として複数サービス（arumono-recipe ほか）を相乗りさせる
- **OCIインスタンス**: Ampere A1 (ARM) を 4 OCPU / 24GB RAM 構成で1台。Always Free枠の上限
- **Kubernetes**: **kubeadm** でシングルノードクラスタを構築
  - `kubeadm init` でコントロールプレーンを立ち上げ、各コンポーネント（kube-apiserver / scheduler / controller-manager / etcd / kubelet / kube-proxy）を個別に管理
  - `kubeadm upgrade` によるバージョンアップ、`kubeadm certs renew` による証明書管理を実地で体験できる
  - ARM (Ampere A1) 公式対応。CNI は Flannel または Calico を別途導入
  - Ingress Controller は組み込まれていないため nginx-ingress を明示的に導入する（学習目的に合致）
- **Namespace 分離**: サービスごとに Namespace を切り、ResourceQuota / LimitRange / NetworkPolicy / RBAC で相互干渉を抑制する
- **Ingress**: `ingress-nginx` を1セット導入し、サブドメイン振り分けで複数サービスをホストする（例: `arumono.example.com`, `another.example.com`）
- **TLS**: `cert-manager` + Let's Encrypt (HTTP-01) で自動更新。複数ホスト名分の証明書を一元管理
- **DNS**: 任意のDNSプロバイダ（Cloudflare等）で各サブドメインを Always Free Public IP に向ける
- **ロードバランサ**: OCI Always Free Load Balancer (10Mbps) を front に置くか、Public IPを直接ノードに向ける構成のどちらでもよい（後段の基本設計で確定）
- **ストレージ**: Block Volume Always Free 枠 (合計200GB) を PV として割り当て

### 4.8 開発・運用ツール

- **リポジトリ戦略**: ポートフォリオ最適化のため分離型を採用。詳細は `docs/repository-strategy.md` を参照
  - `homelab-cluster` ... OCI/Terraform/k8s(kubeadm)/共通基盤/GitOps
  - `arumono-recipe` ... 本サービスのアプリコード + k8s マニフェスト (`deploy/`)
  - 将来追加サービス ... サービスごとに独立リポジトリ
- **コンテナレジストリ**: GitHub Container Registry (Public は無料、Private も個人なら無料枠で十分)
- **CI/CD**: GitHub Actions
  - PR時: lint / type-check / unit test
  - main マージ時: イメージビルド・push、ArgoCD pull型で同期（初期は `kubectl apply` で可）
- **IaC**: Terraform (OCI provider) でインスタンス・ネットワーク、Helmfile / Kustomize で k8s 上のアプリ
- **GitOps**: ArgoCD を `homelab-cluster` で運用し、各サービスリポジトリの `deploy/overlays/prod` を参照
- **監視**: Prometheus + Grafana + Loki + node-exporter を kube-prometheus-stack で導入（クラスタ共通）
- **シークレット**: sealed-secrets でGit管理可能な暗号化Secretへ変換
- **ローカル開発**: Docker Compose で Postgres / MinIO を立てる。Next.js は `pnpm dev`

## 5. システム構成イメージ

```text
                  [ ユーザー (スマホ PWA) ]
                            |
                       HTTPS (TLS)
                            |
              [ OCI Always Free Public IP ]
                            |
                    [ nginx-ingress ]
                            |
        +---------+---------+-----------+
        |                   |           |
   [ Next.js Pod ]     [ MinIO Pod ]  [ Grafana / Prometheus ]
        |                   |
        |                   +---- レシート画像（短期）
        |
        +---- Prisma ---->  [ PostgreSQL (CloudNativePG) ]
        |
        +---- HTTPS ---->   [ Google Cloud ]
                                - Vertex AI Gemini (LLM)
                                - Vertex AI Imagen 3 (画像生成)
                                - Document AI (OCR)
```

## 6. コスト試算（個人利用想定）

| 項目 | 月額目安 | 備考 |
| --- | --- | --- |
| OCI Compute (Ampere A1) | 0円 | Always Free 枠内 |
| OCI Block Storage | 0円 | 200GB Always Free 枠内 |
| OCI Load Balancer | 0円 | 10Mbps Always Free 枠内 |
| OCI Outbound 通信 | 0円 | 月10TB Always Free 枠内 |
| DNS | 0〜数百円 | プロバイダ次第（Cloudflare無料等） |
| Vertex AI Gemini 2.5 Flash | 数百円〜2,000円 | 1日数回の提案を想定 |
| Vertex AI Imagen 3 | 数百円〜数千円 | 1画像 ≒ $0.02〜0.04、キャッシュで抑制 |
| Document AI Expense Parser | 数百円 | 1ページ ≒ $0.10、月数十件想定 |
| **合計（概算）** | **1,000〜5,000円** | AI利用量に依存 |

実利用前にGoogle Cloudの予算アラート（しきい値 ¥3,000 / ¥5,000）を設定する前提。

## 7. リスクと緩和策

| リスク | 緩和策 |
| --- | --- |
| OCI Always Free インスタンスが回収される | クレジットカード登録（Pay-as-you-goに昇格しつつ Always Free を維持）、定期負荷でアイドル判定回避 |
| Ampere A1 がリージョンで枯渇しインスタンス作成に失敗 | リージョン変更、または時間帯を変えて再試行 |
| 1ノード k8s の単一障害点 | 学習目的のため許容。データはCloudNativePGのバックアップを OCI Object Storage に転送 |
| Imagen / Gemini のコスト超過 | 生成画像のキャッシュ、月次予算アラート、`gemini-2.5-flash` 優先利用 |
| 個人情報（レシート画像）のクラウド送信 | アップロード即削除、保存期間は最大24時間、利用規約・プライバシーポリシーで明示 |
| Google サービスアカウントキーの漏洩 | sealed-secrets で暗号化、Workload Identity Federation への移行を将来検討 |

## 8. 代替プラン

| シナリオ | 代替案 |
| --- | --- |
| kubeadm管理の学習比重を下げて早く完成させたい | OCI Compute上で Docker Compose のみ、または Vercel + Supabase に切替 |
| Google系を縛らない | LLMをOpenAI/Anthropicに、OCRをCloud Vision以外（AWS Textract / Azure Document Intelligence）に差し替え |
| 完全無料に振り切りたい | LLMをローカルLLM（Ollama on Ampere A1, 軽量モデル）に置き換え。ただし提案品質は大きく低下 |

## 9. 残課題・今後の確定事項

- 採用するDNSプロバイダの確定
- OCIテナンシのリージョン確定（Ampere A1 在庫を踏まえる）
- GitHub Actions 上で OCI へデプロイする方式（kubeconfig 経由 / OCI CLI / ArgoCD pull型）
- Google Cloud プロジェクト構成（dev / prod の分離方針）
- Workload Identity Federation（OCI→GCP）の利用可否検証
- 監視・アラート通知先（メール / Discord 等）
- バックアップの暗号化と保管先（OCI Object Storage Always Free 枠 20GB の範囲）
