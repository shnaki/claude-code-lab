# claude-code-lab

Claude Code の業務利用を前提にしたセキュリティ設定例を調査・検証するための実験リポジトリです。

## ファイル構成

```
claude-code-lab/
├─ examples/
│  ├─ user-settings.recommended.json    # ユーザー設定・推奨（バランス型）
│  ├─ user-settings.strict.json         # ユーザー設定・最厳格（許可リスト型）
│  ├─ managed-settings.json             # Teamサーバー管理設定（コンソール配布用）
│  └─ organization-instructions.md      # 組織の指示テンプレート（コンソール専用フィールド用）
└─ docs/
   └─ security-settings.md              # 設定の解説・使い方・検証手順
```

## 設定テンプレートの概要

| ファイル | スコープ | 厳格さ | 主な用途 |
|---------|---------|--------|---------|
| `user-settings.recommended.json` | ユーザー設定 | バランス型 | 一般業務開発者向け。危険操作だけブロック |
| `user-settings.strict.json` | ユーザー/プロジェクト設定 | 最厳格型 | 機密プロジェクト向け。許可リスト方式 |
| `managed-settings.json` | マネージド設定 | IT強制 | **Team サーバー管理設定**。コンソールから全メンバーへ一括配布 |
| `organization-instructions.md` | 組織の指示 | ソフト制御 | コンソール「組織の指示」専用フィールドに貼付。settings.json で表現できない振る舞い・方針の補完 |

## 共通セキュリティ方針

セキュリティ制御を **二層構成** で設計しています。

- **ハード制御**（`managed-settings.json`）: ツール・コマンド単位の強制ブロック。
  ユーザーが許可しても自動実行できない操作を定義します。
- **ソフト制御**（`organization-instructions.md` / 組織の指示フィールド）: 振る舞い・コーディング方針を自然言語で定義。
  シークレット取り扱い、プロンプトインジェクション対策、セキュアコーディング原則など、
  settings.json では表現できない領域を補完します。

すべての JSON テンプレートで以下も採用しています。

**通信制御（ハイブリッド）**
- Anthropic へのテレメトリ・自動更新・エラー報告は停止
- 監査ログは組織の OTEL コレクタへ送信（placeholder → 実値に変更が必要）

**deny 対象（4カテゴリ）**
- 機密ファイル（`.env`, 秘密鍵, クラウド認証情報）の読み取り
- 外部ダウンロード・送信（`curl`, `wget`, `Invoke-WebRequest` 等）
- 破壊的コマンド（`rm -rf`, `git push --force`, `git reset --hard` 等）
- コード実行・権限昇格（`sudo`, `runas`, `npm publish`, `docker push` 等）

## 使い方

詳細は [`docs/security-settings.md`](docs/security-settings.md) を参照してください。

**Team サーバー管理設定（`managed-settings.json`）の配布手順**:

1. `forceLoginOrgUUID` を組織 UUID に置き換える（または削除）
2. `OTEL_EXPORTER_OTLP_ENDPOINT` を自社コレクタ URL に置き換える
3. Claude.ai 管理コンソール → **Admin Settings → Claude Code → Managed settings** に JSON を貼付・保存
4. メンバーが Claude Code を再起動すると設定が反映される（`/status` / `/permissions` で確認）

**組織の指示（`organization-instructions.md`）の配布手順**:

1. Claude.ai 管理コンソール → **Admin Settings → Claude Code** を開く
2. **「組織の指示（Organization Instructions）」フィールド** に `organization-instructions.md` の本文を貼付・保存

**ユーザー設定（推奨型）を個人設定に適用する場合**:

```powershell
# バックアップ
Copy-Item "$env:USERPROFILE\.claude\settings.json" "$env:USERPROFILE\.claude\settings.json.bak"

# テンプレートを適用
Copy-Item "examples\user-settings.recommended.json" "$env:USERPROFILE\.claude\settings.json"
```

## 注意事項

- `managed-settings.json` の `forceLoginOrgUUID` は **placeholder** です。実際の組織 UUID に置き換えるか、キーごと削除してください。
- `OTEL_EXPORTER_OTLP_ENDPOINT` も **placeholder** です。実値に差し替えてから配布してください。
- `managed-settings.json` は **Claude for Teams（v2.1.38 以降）** のサーバー管理設定として設計されています。endpoint-managed（ファイル配置/MDM）として使う場合はドキュメントを参照してください。
- `managed-settings.json` の `forceRemoteSettingsRefresh: true` により、`api.anthropic.com` に到達できない環境ではユーザーが起動できなくなります。ファイアウォール設定を確認してください。
- `organization-instructions.md`（組織の指示）は **Claude for Teams（v2.1.38 以降）** または **Claude for Enterprise（v2.1.30 以降）** でのみ利用可能です。Bedrock / Vertex AI 経由では使用できません。
- このリポジトリ自体の変更は個人の `~/.claude/settings.json` には影響しません。
