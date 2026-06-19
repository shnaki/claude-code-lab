# claude-code-lab

Claude Code の業務利用を前提にしたセキュリティ設定例を調査・検証するための実験リポジトリです。

## ファイル構成

```
claude-code-lab/
├─ examples/
│  ├─ user-settings.recommended.json    # ユーザー設定・推奨（バランス型）
│  ├─ user-settings.strict.json         # ユーザー設定・最厳格（許可リスト型）
│  ├─ managed-settings.json             # マネージド設定（IT強制レイヤー）
│  └─ organization-instructions.md      # 組織の指示テンプレート（claudeMd 用）
└─ docs/
   └─ security-settings.md              # 設定の解説・使い方・検証手順
```

## 設定テンプレートの概要

| ファイル | スコープ | 厳格さ | 主な用途 |
|---------|---------|--------|---------|
| `user-settings.recommended.json` | ユーザー設定 | バランス型 | 一般業務開発者向け。危険操作だけブロック |
| `user-settings.strict.json` | ユーザー/プロジェクト設定 | 最厳格型 | 機密プロジェクト向け。許可リスト方式 |
| `managed-settings.json` | マネージド設定 | IT強制 | 組織全体ポリシー。管理者権限で配置 |
| `organization-instructions.md` | 組織の指示 (claudeMd) | ソフト制御 | settings.json で表現できない振る舞い・方針の補完 |

## 共通セキュリティ方針

セキュリティ制御を **二層構成** で設計しています。

- **ハード制御**（`settings.json` / `managed-settings.json`）: ツール・コマンド単位の強制ブロック。
  ユーザーが許可しても自動実行できない操作を定義します。
- **ソフト制御**（`organization-instructions.md` / `claudeMd`）: 振る舞い・コーディング方針を自然言語で定義。
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

**クイックスタート**（推奨型を個人設定に適用する場合）:

```powershell
# バックアップ
Copy-Item "$env:USERPROFILE\.claude\settings.json" "$env:USERPROFILE\.claude\settings.json.bak"

# テンプレートを適用
Copy-Item "examples\user-settings.recommended.json" "$env:USERPROFILE\.claude\settings.json"
```

## 注意事項

- `examples/managed-settings.json` の `forceLoginMethod` / `forceLoginOrgUUID` / `OTEL_EXPORTER_OTLP_ENDPOINT` は **placeholder** です。実値に差し替えてから使用してください。
- このリポジトリ自体の変更は個人の `~/.claude/settings.json` には影響しません。
- マネージド設定の実機配置には管理者権限が必要です。
- `examples/organization-instructions.md`（`claudeMd` 用）は **Claude for Teams（v2.1.38 以降）** または **Claude for Enterprise（v2.1.30 以降）** でのみ利用可能です。Bedrock / Vertex AI 経由では使用できません。
