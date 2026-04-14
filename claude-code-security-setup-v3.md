# Claude Code セキュリティ自動設定プロンプト v3

> **使い方**: 「このプロンプトをClaude Codeに貼り付けてください」セクションの中身をClaude Codeに貼り付けて実行してください。
> サンドボックス・フック・ルール・パーミッションが自動で設定されます。

---

## このプロンプトをClaude Codeに貼り付けてください

```
以下の手順でClaude Codeのセキュリティ設定を行ってください。
すべてのファイルを正確に作成・更新してください。省略や要約はしないでください。

---

## 前提条件の確認

まず以下を確認してください:
1. jq がインストールされているか確認（`which jq`）。なければ `brew install jq` を実行
2. gitleaks がインストールされているか確認（`which gitleaks`）。なければ `brew install gitleaks` を実行
3. `~/.claude/hooks/` ディレクトリが存在するか確認。なければ `mkdir -p ~/.claude/hooks` で作成

---

## STEP 0: サンドボックスの確認と推奨設定

### サンドボックスとは
AIが実行する **Bashコマンドとその子プロセス** をOS レベルで物理的に制限する仕組みです。
フックやルールは「やらないでね」というお願いですが、サンドボックスは「やれない」状態を作ります。

**重要: サンドボックスの対象と対象外**
- 対象: Bashツール（シェルコマンド）とその子プロセス
- 対象外: Read / Edit / Write ツール（これらはパーミッション設定で制御する）

→ サンドボックスだけでは不十分。パーミッション（deny）やフックとの併用が必須です。

### 有効化方法

**方法1: セッション内コマンド**
Claude Code を起動した後、プロンプトに以下を入力:
```
/sandbox
```
メニューが表示され、以下から選択できます:
- Auto-allow mode: サンドボックス内のbashコマンドを自動許可（権限プロンプトなし）
- Regular permissions mode: サンドボックス内でも権限フローを通す（より厳密）

**方法2: settings.json で常時有効化（推奨）**
`~/.claude/settings.json` に以下を追加:
```json
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "filesystem": {
      "allowWrite": ["/tmp"],
      "denyRead": ["~/.aws/credentials", "~/.ssh"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"]
    }
  }
}
```

**設定の重要ポイント:**
- `allowUnsandboxedCommands: false` — これを設定しないと、サンドボックスで失敗したコマンドが非サンドボックスで再実行されてしまう（デフォルトはtrue = 穴）
- `denyRead` — 機密ファイルの読み取りも制限できる
- `allowedDomains` — ネットワーク通信をドメイン単位で制御できる

### サンドボックスの仕組み
- **macOS**: Seatbelt フレームワークを使用（追加インストール不要）
- **Linux/WSL2**: bubblewrap + socat を使用
  ```bash
  # Ubuntu/Debian
  sudo apt-get install bubblewrap socat
  # Fedora
  sudo dnf install bubblewrap socat
  ```
- **WSL1**: サポートされていない
- **Windows**: ネイティブサポートは未実装（今後対応予定）

### Docker内での実行（最も堅牢）
Docker内でClaude Codeを動かすとOS レベルで完全に隔離できます:

```bash
# Step 1: Claude Code入りのイメージをビルド
docker build -t claude-sandbox -f - . <<'DOCKERFILE'
FROM node:20
RUN npm install -g @anthropic-ai/claude-code
WORKDIR /workspace
DOCKERFILE

# Step 2: ネットワーク遮断して実行（必要なファイルだけマウント）
docker run -it --rm \
  -v $(pwd):/workspace \
  -w /workspace \
  --network=none \
  claude-sandbox claude
```

`--network=none` でネットワークを完全遮断、`-v` でマウントするディレクトリを限定できます。
※ インストールはビルド時（Step 1）にネットワーク接続下で行い、実行時（Step 2）にネットワークを遮断します。

### 重要な理解
サンドボックスとフック/ルールは役割が違います。両方あるのが理想です:

| | サンドボックス | フック/ルール | パーミッション（deny） |
|---|---|---|---|
| 対象 | Bashコマンドのみ | 全ツール | Read/Edit/Write |
| 制限レベル | OS・カーネル（物理的） | アプリケーション（論理的） | アプリケーション（強制） |
| バイパス | 不可能 | 工夫すれば可能 | 不可能 |
| ネットワーク制限 | できる | できない | できない |
| ファイル読み取り制限 | できる（denyRead） | できない | できない |

→ サンドボックス = Bashコマンドの物理的制限
→ パーミッション（deny） = Read/Edit/Writeの強制拒否
→ フック/ルール = うっかりミスの防止・パターン検出

---

## STEP 1: フックスクリプトの作成

以下の4つのシェルスクリプトを `~/.claude/hooks/` に作成してください。
※ ファイル名に "secret" や "token" が含まれる場合、deny ルールに引っかかる可能性があります。
※ その場合は `/tmp/` に一時ファイルを作成してから `cp` で移動してください。

### 1-1. ~/.claude/hooks/content-secret-scan.sh

APIキー・トークンのハードコードを機械的にブロックするフック。
Edit/Write の実行前に発火し、シークレットが検出されたら exit 2 でブロックします。

```bash
#!/bin/bash
# Claude Code Hook: Edit/Writeされるファイルの中身にシークレットが含まれていないかチェック
# PreToolUse (Edit|Write) で発火
# exit 0 = 許可, exit 2 = ブロック

INPUT=$(cat)

if command -v jq &>/dev/null; then
  TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty' 2>/dev/null)
  CONTENT=$(echo "$INPUT" | jq -r '.tool_input.new_string // .tool_input.content // empty' 2>/dev/null)
  FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)
else
  TOOL_NAME=$(echo "$INPUT" | grep -o '"tool_name"[[:space:]]*:[[:space:]]*"[^"]*"' | head -1 | sed 's/.*"tool_name"[[:space:]]*:[[:space:]]*"//;s/"$//')
  CONTENT=$(echo "$INPUT" | grep -o '"new_string"[[:space:]]*:[[:space:]]*"[^"]*"' | head -1 | sed 's/.*"new_string"[[:space:]]*:[[:space:]]*"//;s/"$//')
  FILE_PATH=$(echo "$INPUT" | grep -o '"file_path"[[:space:]]*:[[:space:]]*"[^"]*"' | head -1 | sed 's/.*"file_path"[[:space:]]*:[[:space:]]*"//;s/"$//')
fi

[ -z "$CONTENT" ] && exit 0

# テストファイル、ドキュメント、設定例ファイルは除外（パスベースで精密に）
case "$FILE_PATH" in
  */test/*|*/tests/*|*/__tests__/*|*.test.*|*.spec.*|*mock*|*.example|*.sample|*.template|*.md|*.mdx|*.txt|*.rst)
    exit 0
    ;;
esac

ISSUES=""

# ============================================================
# パターンA: サービス固有のAPIキー形式（確実にブロック）
# ============================================================

echo "$CONTENT" | grep -qE 'AKIA[0-9A-Z]{16}' && ISSUES="$ISSUES\n  AWS Access Key detected"
echo "$CONTENT" | grep -qE '(aws_secret_access_key|AWS_SECRET_ACCESS_KEY)\s*[:=]\s*[A-Za-z0-9/+=]{40}' && ISSUES="$ISSUES\n  AWS Secret Key detected"
echo "$CONTENT" | grep -qE '(ghp_|gho_|ghs_|ghr_|github_pat_)[A-Za-z0-9_]{20,}' && ISSUES="$ISSUES\n  GitHub Token detected"
echo "$CONTENT" | grep -qE 'xox[bpsa]-[A-Za-z0-9\-]{20,}' && ISSUES="$ISSUES\n  Slack Token detected"
echo "$CONTENT" | grep -qE 'AIza[A-Za-z0-9\-_]{35}' && ISSUES="$ISSUES\n  Google API Key detected"
echo "$CONTENT" | grep -qE 'sk-ant-[A-Za-z0-9\-_]{20,}' && ISSUES="$ISSUES\n  Anthropic API Key detected"
echo "$CONTENT" | grep -qE 'sk-proj-[A-Za-z0-9\-_]{20,}' && ISSUES="$ISSUES\n  OpenAI API Key detected"
echo "$CONTENT" | grep -qE 'sk-[A-Za-z0-9]{40,}' && ISSUES="$ISSUES\n  OpenAI-style API Key detected"
echo "$CONTENT" | grep -qE 'AIzaSy[A-Za-z0-9\-_]{33}' && ISSUES="$ISSUES\n  Google Cloud/Gemini API Key detected"
echo "$CONTENT" | grep -qiE '(freee|FREEE).*[Tt]oken\s*[:=]\s*["'"'"'"][A-Za-z0-9]{20,}' && ISSUES="$ISSUES\n  freee API Token detected"
echo "$CONTENT" | grep -qE '_authToken\s*=\s*[A-Za-z0-9\-_]{20,}' && ISSUES="$ISSUES\n  npm auth token detected"
echo "$CONTENT" | grep -qE '\-\-\-\-\-BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY\-\-\-\-\-' && ISSUES="$ISSUES\n  Private Key block detected"
echo "$CONTENT" | grep -qE '(sk_live_|pk_live_|sk_test_|rk_live_)[A-Za-z0-9]{20,}' && ISSUES="$ISSUES\n  Stripe Key detected"
echo "$CONTENT" | grep -qE 'SG\.[A-Za-z0-9\-_]{20,}\.[A-Za-z0-9\-_]{20,}' && ISSUES="$ISSUES\n  SendGrid API Key detected"
echo "$CONTENT" | grep -qE 'AC[a-f0-9]{32}' && ISSUES="$ISSUES\n  Twilio Account SID detected"

# ============================================================
# パターンB: 変数名ベースのハードコード検出（最重要）
# ============================================================

EXCLUDE_PAT='(process\.env|os\.environ|os\.getenv|ENV\[|getenv|PropertiesService|System\.getenv|config\.|settings\.|\.get\(|placeholder|example|your[_-]|xxx|changeme|TODO|FIXME|REPLACE|INSERT|<.*>|\$\{|\{\{|dummy|sample|test[_-]?key|fake|mock|MOCK|DUMMY|EXAMPLE|PLACEHOLDER|YOUR_)'

# key=value 形式（8文字以上の値）
if echo "$CONTENT" | grep -iE '(api[_-]?key|secret[_-]?key|access[_-]?key|private[_-]?key|auth[_-]?key|api[_-]?secret|client[_-]?secret|app[_-]?secret|api[_-]?token|access[_-]?token|auth[_-]?token|bearer[_-]?token|refresh[_-]?token|password|passwd|credential)\s*[:=]\s*["'"'"'`][A-Za-z0-9/+=\-_.]{8,}["'"'"'`]' \
   | grep -ivE "$EXCLUDE_PAT" \
   | head -1 | grep -q .; then
  ISSUES="$ISSUES\n  HARDCODED SECRET: APIキー・トークンが直接コードに書かれています。環境変数を使ってください"
fi

# const/let/var/def による直接代入
if echo "$CONTENT" | grep -iE '(const|let|var|def|private|public|static)\s+\w*(key|secret|token|password|credential|auth)\w*\s*=\s*["'"'"'`][A-Za-z0-9/+=\-_.]{8,}["'"'"'`]' \
   | grep -ivE "$EXCLUDE_PAT" \
   | head -1 | grep -q .; then
  ISSUES="$ISSUES\n  HARDCODED SECRET (変数宣言): APIキーを直書きしています。環境変数を使ってください"
fi

# ============================================================
# パターンC: Bearer/Authorization ヘッダーに直書き
# ============================================================
if echo "$CONTENT" | grep -iE '(Authorization|Bearer)\s*[:=]\s*["'"'"'](Bearer\s+)?[A-Za-z0-9/+=\-_.]{20,}["'"'"']' \
   | grep -ivE "$EXCLUDE_PAT" \
   | head -1 | grep -q .; then
  ISSUES="$ISSUES\n  HARDCODED AUTH: Authorization/Bearerヘッダーにトークンが直書きされています"
fi

if [ -n "$ISSUES" ]; then
  echo -e "BLOCKED - シークレット検出 in $FILE_PATH:$ISSUES" >&2
  echo -e "\n修正方法: 環境変数（.env / PropertiesService / process.env）からキーを読み込んでください。コードに直接書かないでください。" >&2
  exit 2
fi

exit 0
```

### 1-2. ~/.claude/hooks/secret-scan.sh

git commit/push 時にシークレットをスキャンするフック。

```bash
#!/bin/bash
# Claude Code Hook: Scan staged files for secrets before git commit/push
# exit 0 = allow, exit 2 = block

INPUT=$(cat)

if command -v jq &>/dev/null; then
  COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
else
  COMMAND=$(echo "$INPUT" | grep -o '"command"[[:space:]]*:[[:space:]]*"[^"]*"' | head -1 | sed 's/.*"command"[[:space:]]*:[[:space:]]*"//;s/"$//')
fi

case "$COMMAND" in
  git\ commit*|git\ push*)
    ;;
  *)
    exit 0
    ;;
esac

# --- gitleaks による自動スキャン（git push時） ---
case "$COMMAND" in
  git\ push*)
    if command -v gitleaks &>/dev/null; then
      GITLEAKS_OUTPUT=$(gitleaks detect --no-git --no-banner 2>&1)
      GITLEAKS_EXIT=$?
      if [ $GITLEAKS_EXIT -ne 0 ]; then
        echo "gitleaks detected secrets - blocking git push:" >&2
        echo "$GITLEAKS_OUTPUT" >&2
        exit 2
      fi
    else
      echo "WARNING: gitleaks が未インストールです。push時のシークレットスキャンが無効になっています。brew install gitleaks を実行してください。" >&2
    fi
    ;;
esac

STAGED_FILES=$(git diff --cached --name-only 2>/dev/null)

if [ -z "$STAGED_FILES" ]; then
  exit 0
fi

ISSUES=""

for file in $STAGED_FILES; do
  case "$file" in
    .env|.env.*|*.env)
      ISSUES="$ISSUES\n  BLOCKED: .env file staged: $file"
      ;;
    *.pem|*.key|*.p12|*.pfx)
      ISSUES="$ISSUES\n  BLOCKED: Private key/cert staged: $file"
      ;;
    *secret*|*credential*|*service-account*)
      ISSUES="$ISSUES\n  BLOCKED: Sensitive file staged: $file"
      ;;
    .aws/*|.ssh/*)
      ISSUES="$ISSUES\n  BLOCKED: Cloud/SSH config staged: $file"
      ;;
  esac
done

for file in $STAGED_FILES; do
  if [ ! -f "$file" ]; then
    continue
  fi

  if file "$file" | grep -q "binary"; then
    continue
  fi

  SECRETS=$(git diff --cached -- "$file" | grep -iE '^\+.*(PRIVATE KEY|api_key|apikey|secret_key|password|access_token|auth_token)\s*[:=]' | grep -v '^\+\+\+' | head -5)
  if [ -n "$SECRETS" ]; then
    ISSUES="$ISSUES\n  WARNING: Possible secret in $file"
  fi
done

if [ -n "$ISSUES" ]; then
  echo -e "Secret scan found issues:$ISSUES" >&2
  exit 2
fi

exit 0
```

### 1-3. ~/.claude/hooks/command-guard.sh

危険なコマンドパターンを検出して警告するフック（ブロックはしない）。

```bash
#!/bin/bash
# 統合コマンドガード: 危険なコマンドパターンを検出して警告（ブロックはしない）
INPUT=$(cat)
if command -v jq &>/dev/null; then
  COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
else
  COMMAND=$(echo "$INPUT" | grep -o '"command"[[:space:]]*:[[:space:]]*"[^"]*"' | head -1 | sed 's/.*"command"[[:space:]]*:[[:space:]]*"//;s/"$//')
fi

[ -z "$COMMAND" ] && exit 0

WARNING=""

# --- npm install ---
case "$COMMAND" in
  npm\ install*|npm\ i\ *)
    case "$COMMAND" in *--ignore-scripts*) ;; *)
      WARNING="npm install が実行されます。リスク: postinstallスクリプトが自動実行され、悪意あるコードが動く可能性があります。対策: (1) npm ci --ignore-scripts が安全 (2) npm audit を実行 (3) パッケージのDL数・更新日を確認"
    ;; esac ;;
esac

# --- pip install ---
[ -z "$WARNING" ] && case "$COMMAND" in
  pip\ install*|pip3\ install*|python*-m\ pip\ install*)
    case "$COMMAND" in *--no-deps*|*--require-hashes*) ;; *)
      WARNING="pip install が実行されます。リスク: setup.pyが自動実行されます。対策: (1) pip install --no-deps (2) pip audit を実行 (3) requirements.txtに固定バージョンで記録"
    ;; esac ;;
esac

# --- npx ---
[ -z "$WARNING" ] && case "$COMMAND" in
  npx\ *)
    WARNING="npx はパッケージをDLして即座に実行します。リスク: typosquatting。対策: パッケージ名のスペルを確認"
    ;;
esac

# --- curl | bash ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *curl*\|*bash*|*curl*\|*sh*|*curl*\|*zsh*|*wget*\|*bash*|*wget*\|*sh*|*wget*\|*zsh*)
    WARNING="URLからDLしたスクリプトを直接実行しようとしています。まずDLして中身を確認してください"
    ;;
esac

# --- git push --force ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *push*--force*|*push*-f\ *|*push*-f$)
    WARNING="git push --force が実行されます。安全な代替: git push --force-with-lease"
    ;;
esac

# --- git add . / git add -A ---
[ -z "$WARNING" ] && case "$COMMAND" in
  git\ add\ .|git\ add\ -A*|git\ add\ --all*)
    WARNING="git add . / git add -A が実行されます。リスク: .env等が意図せずステージングされます。git add <ファイル名> で個別に追加してください"
    ;;
esac

# --- sudo ---
[ -z "$WARNING" ] && case "$COMMAND" in
  sudo\ *)
    WARNING="sudo（管理者権限）で実行しようとしています。本当にsudoが必要ですか？"
    ;;
esac

# --- chmod 777 ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *chmod*777*|*chmod*a+rwx*)
    WARNING="chmod 777 が実行されます。適切なパーミッション（644/755/600）を使ってください"
    ;;
esac

# --- docker run ---
[ -z "$WARNING" ] && case "$COMMAND" in
  docker\ run*)
    WARNING="docker run が実行されます。-v /:/host や --privileged は危険です"
    ;;
esac

# --- 環境変数の外部送信 ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *curl*\$*|*curl*\${*|*wget*--post*\$*)
    case "$COMMAND" in
      *API_KEY*|*SECRET*|*TOKEN*|*PASSWORD*|*CREDENTIAL*)
        WARNING="環境変数を含むHTTPリクエストが実行されます。本番のキーが外部に送信されるリスクがあります"
        ;;
    esac ;;
esac

# --- curl/wget でファイル送信 ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *curl*--data*@*|*curl*-d\ @*|*curl*--upload*|*curl*-T\ *|*wget*--post-file*)
    WARNING="ローカルファイルを外部サーバーに送信しようとしています。機密ファイルが含まれていないか確認してください"
    ;;
esac

# --- rm -rf 危険パターン ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *rm\ *-*r*-*f*|*rm\ *-*f*-*r*|*rm\ *-rf*|*rm\ *-fr*)
    case "$COMMAND" in
      *\~/*|*\$HOME*|*/Users/*|*\.\./*)
        WARNING="rm -rf がホームディレクトリまたは親ディレクトリに対して実行されます。復元不可です。"
        ;;
    esac ;;
esac

# --- eval ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *eval\ *\$*|*eval\ *\`*)
    WARNING="eval でシェル変数を展開しようとしています。任意コード実行のリスクがあります"
    ;;
esac

# --- base64 デコード+実行 ---
[ -z "$WARNING" ] && case "$COMMAND" in
  *base64*-d*\|*bash*|*base64*-d*\|*sh*|*base64*--decode*\|*bash*|*base64*--decode*\|*sh*)
    WARNING="base64デコードした内容をシェルで実行しようとしています。難読化された悪意あるコードの可能性があります"
    ;;
esac

# --- git config --global ---
[ -z "$WARNING" ] && case "$COMMAND" in
  git\ config\ --global*)
    WARNING="git config --global はすべてのリポジトリに影響します。--local で十分でないか確認してください"
    ;;
esac

# --- ngrok / localtunnel ---
[ -z "$WARNING" ] && case "$COMMAND" in
  ngrok\ *|lt\ --port*|cloudflared\ tunnel*)
    WARNING="ローカルサーバーをインターネットに公開しようとしています"
    ;;
esac

if [ -n "$WARNING" ]; then
  echo "{\"hookSpecificOutput\": {\"hookEventName\": \"PreToolUse\", \"additionalContext\": \"WARNING: $WARNING\"}}"
fi
exit 0
```

### 1-4. ~/.claude/hooks/env-gitignore-check.sh

.env ファイルが作成されたとき、.gitignore に .env が含まれているか確認するフック。

```bash
#!/bin/bash
# Claude Code Hook: PostToolUse Write - .envファイルが作成されたとき .gitignore を確認

INPUT=$(cat)

if command -v jq &>/dev/null; then
  FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)
else
  FILE_PATH=$(echo "$INPUT" | grep -o '"file_path"[[:space:]]*:[[:space:]]*"[^"]*"' | head -1 | sed 's/.*"file_path"[[:space:]]*:[[:space:]]*"//;s/"$//')
fi

case "$(basename "$FILE_PATH")" in
  .env|.env.*)
    ;;
  *)
    exit 0
    ;;
esac

DIR=$(dirname "$FILE_PATH")
GITIGNORE=""

while [ "$DIR" != "/" ] && [ "$DIR" != "." ]; do
  if [ -f "$DIR/.gitignore" ]; then
    GITIGNORE="$DIR/.gitignore"
    break
  fi
  if [ -d "$DIR/.git" ]; then
    break
  fi
  DIR=$(dirname "$DIR")
done

WARNING=""

if [ -z "$GITIGNORE" ]; then
  WARNING=".envファイルが作成されましたが、.gitignoreが見つかりません。.gitignoreを作成して .env を追加してください。"
elif ! grep -qE '^\s*\.env\s*$|^\s*\.env\.\*\s*$' "$GITIGNORE" 2>/dev/null; then
  WARNING=".envファイルが作成されましたが、.gitignore に .env パターンが含まれていません。"
fi

if [ -n "$WARNING" ]; then
  echo "{\"hookSpecificOutput\": {\"hookEventName\": \"PostToolUse\", \"additionalContext\": \"WARNING: $WARNING\"}}"
fi

exit 0
```

すべてのスクリプトに `chmod +x` を実行してください。

---

## STEP 2: settings.json の更新

`~/.claude/settings.json` を以下の内容で作成または更新してください。
既存の settings.json がある場合は、`permissions` と `hooks` セクションをマージしてください。

```json
{
  "permissions": {
    "allow": [],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(rm -r /)",
      "Bash(dd:*)",
      "Bash(mkfs:*)",
      "Edit(.env*)",
      "Edit(*.pem)",
      "Edit(*.key)",
      "Edit(*.p12)",
      "Edit(*secret*)",
      "Edit(*credential*)",
      "Edit(*token*)",
      "Edit(*service-account*)",
      "Edit(.ssh/*)",
      "Edit(.aws/*)",
      "Write(.env*)",
      "Write(*.pem)",
      "Write(*.key)",
      "Write(*.p12)",
      "Write(*secret*)",
      "Write(*credential*)",
      "Write(*token*)",
      "Write(*service-account*)",
      "Write(.ssh/*)",
      "Write(.aws/*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/secret-scan.sh"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/content-secret-scan.sh"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/command-guard.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/env-gitignore-check.sh",
            "timeout": 5,
            "statusMessage": ".envファイルの安全性を確認中..."
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[SECURITY RULES RE-INJECTED] 最優先: APIキー・トークン・パスワードをコードに直書きするな。絶対に。process.env/PropertiesService/os.environを使え。 1.eval()禁止 2.エラー握りつぶし禁止 3.ログにPII出力禁止 4.外部APIにタイムアウト設定 5..gitignoreに.env,*.key,*.pem含める 6.AI生成コードの外部通信は送信先を明示 7.破壊的操作は事前確認必須 8.新規パッケージはDL数・脆弱性を確認'"
          }
        ]
      }
    ]
  }
}
```

### settings.local.json の注意事項

`~/.claude/settings.local.json` は、ツール実行時に「Always allow」を選択すると allow が蓄積されていきます。
以下のパターンが allow に含まれている場合、`bash -c "任意のコマンド"` のようにdenyルールを迂回できてしまいます。**定期的に確認して削除してください**:

| 危険な allow パターン | リスク | 対応 |
|---|---|---|
| `Bash(bash:*)` | `bash -c "rm -rf ..."` のようにdeny迂回可能 | 削除必須 |
| `Bash(cat:*)` | 機密ファイルの中身が読める | 削除推奨 |
| `Bash(python3:*)` / `Bash(python:*)` | 任意のPythonコード実行可能 | 削除推奨 |
| `Bash(npx:*)` | 任意パッケージをDL・即実行可能 | 削除推奨 |
| `Bash(node -e:*)` | 任意のJSコード実行可能 | 削除推奨 |
| `Bash(curl:*)` | 任意URLへの通信が許可 | 削除推奨 |

確認コマンド:
```bash
cat ~/.claude/settings.local.json | python3 -c "
import json,sys
d=json.load(sys.stdin)
dangerous=['Bash(bash:*)', 'Bash(cat:*)', 'Bash(python3:*)', 'Bash(python:*)', 'Bash(npx:*)', 'Bash(node -e:*)', 'Bash(curl:*)']
allows=d.get('permissions',{}).get('allow',[])
found=[a for a in allows if a in dangerous]
if found:
    print('WARNING: 以下の危険な allow を削除してください:')
    for f in found: print(f'  - {f}')
else:
    print('OK: 危険な allow パターンは見つかりませんでした')
"
```

---

## STEP 3: CLAUDE.md にセキュリティルールを追加

`~/.claude/CLAUDE.md` はClaude Code全体に適用されるグローバルルールファイルです。
- ファイルが存在しない場合: `~/.claude/CLAUDE.md` を新規作成し、以下の内容を書き込んでください
- ファイルが既にある場合: 末尾に以下のセキュリティセクションを追加してください
- 既にセキュリティルールが書かれている場合: 既存のセキュリティセクションを以下で置き換えてください

```markdown
# セキュリティルール

## 最優先: APIキー・シークレットの直書き絶対禁止

**これが最も重要なルール。何があっても破らないで。**

- APIキー・パスワード・トークン・秘密鍵をソースコードに直接書くな。絶対に。1文字たりとも。
- 「一時的に」「テスト用に」「あとで変える」も全部ダメ。例外なし。
- 正しい方法:
  - Node.js → `process.env.API_KEY`（`.env` + dotenv）
  - GAS → `PropertiesService.getScriptProperties().getProperty('API_KEY')`
  - Python → `os.environ['API_KEY']`
- `.env.example` にはプレースホルダー（`YOUR_API_KEY_HERE`）だけ書く。実際の値は禁止。
- レポート・ログ・コメント・コミットメッセージにも実際のキー値を書くな。必ずマスキング（`sk-***`）。

## コード生成時のルール（5項目）

1. `eval()` 禁止。`dangerouslySetInnerHTML` 禁止。外部入力をコードとして実行しない。
2. エラーを握りつぶさない（`catch` で何もしない、`except: pass` 禁止）。最低限ログに出力する。
3. ログに個人情報（メールアドレス、電話番号、トークン等）を出力しない。マスキングする。
4. 外部APIへのリクエストにはタイムアウトを設定する。
5. `.gitignore` にプロジェクト作成時に以下を含める: `.env`, `.env.*`, `*.key`, `*.pem`, `credentials*.json`, `service-account*.json`, `node_modules/`, `.venv/`

## AIエージェント対策（3項目）

6. 生成したコードが外部URLに通信する場合、送信先と送信内容を明示する。
7. 破壊的操作（git push, デプロイ, データ削除, rm -rf）は実行前に必ずユーザーに確認する。
8. MCP・外部ツールを新規接続する際は、接続先・付与する権限・アクセス範囲を確認する。

## パッケージ管理（2項目）

9. 新規パッケージ追加前に確認: 週間DL数、最終更新日、`npm audit` / `pip audit`。標準APIで代替できないか検討する。
10. lockファイル（`package-lock.json`, `yarn.lock`, `poetry.lock`）は必ずgitにcommitする。
```

---

## STEP 4: 動作確認

1. **APIキー直書きブロック**: `/tmp/test-security.js` に `const apiKey = "sk-ant-test1234abcd5678efgh"` と書こうとして、ブロックされることを確認
2. **gitleaks**: `which gitleaks` で存在を確認。なければ `brew install gitleaks` を提案
3. **jq**: `which jq` で存在を確認。なければ `brew install jq` を提案
4. **settings.local.json チェック**: 上記の確認コマンドを実行して危険な allow がないか確認

テスト後、`/tmp/test-security.js` は削除してください。

設定完了後、以下のサマリを表示してください:

---
セキュリティ設定完了

【第0層: サンドボックス（OS レベル — Bashコマンドのみ対象）】
- /sandbox コマンド、または settings.json に "sandbox": {"enabled": true} で有効化
- 重要: allowUnsandboxedCommands を false に設定すること（デフォルトtrue = 穴）
- macOS: Seatbelt、Linux: bubblewrap + socat を使用
- Read/Edit/Writeツールは対象外 → パーミッション（deny）で制御する
- Docker内実行が最も確実な隔離方法

【第1層: フック（自動強制）】
- content-secret-scan.sh: APIキー直書きをブロック（15種類のキーパターン + 変数名ベース検出）
- secret-scan.sh: git commit/push時のシークレットスキャン（gitleaks未インストール時は警告）
- command-guard.sh: 危険コマンドの警告（npm/pip/sudo/eval/rm -rf等、16パターン）
- env-gitignore-check.sh: .env作成時の.gitignore確認

【第2層: パーミッション（直接拒否）】
- .env, .pem, .key, .p12, secret, credential, token, service-account, .ssh, .aws のEdit/Writeを拒否
- rm -rf, dd, mkfs の実行を拒否
- settings.local.json の危険な allow パターンを削除済み

【第3層: ルール（CLAUDE.md）】
- APIキー直書き禁止（最優先）
- コード生成5項目 + AIエージェント対策3項目 + パッケージ管理2項目

【補助: SessionStart】
- コンテキスト圧縮後もセキュリティルールを自動再注入
---
```

---

## 構成の全体図

```
セキュリティ 4層構造 + 補助

  【第0層】サンドボックス（OSレベル — Bashコマンドの物理的制限）
   ├ /sandbox コマンド       → Seatbelt/bubblewrapでOS レベル制限
   ├ settings.json設定       → "sandbox": {"enabled": true} で常時有効化
   ├ allowUnsandboxedCommands: false → 失敗時の非サンドボックス再実行を防止
   └ Docker内実行            → 最も堅牢な隔離（--network=none）
   ※ 対象はBashコマンドのみ。Read/Edit/Writeは対象外（第2層で制御）

  【第1層】フック（自動強制 — うっかりミスの防止）
   ├ content-secret-scan.sh  → Edit/Write時にAPIキー検出 → ブロック
   ├ secret-scan.sh          → git commit/push時にシークレット検出 → ブロック
   ├ command-guard.sh        → 危険コマンド検出 → 警告（ブロックしない）
   └ env-gitignore-check.sh  → .env作成時の.gitignore確認 → 警告

  【第2層】パーミッション（settings.json deny — ファイル保護）
   ├ 機密ファイルのEdit/Write直接拒否
   ├ 破壊コマンド(rm -rf/dd/mkfs)の実行拒否
   └ settings.local.json の定期清掃（危険なallow削除）

  【第3層】ルール（CLAUDE.md — 判断が必要なもの）
   └ 12項目に凝縮したセキュリティルール

  【補助】SessionStart再注入
   └ コンテキスト圧縮後にルールを自動再注入
```

## 各層の役割と限界

| 層 | 対象 | 守れるもの | 守れないもの |
|---|---|---|---|
| サンドボックス | Bashコマンドのみ | ネットワーク外部送信、プロジェクト外ファイル操作、未知のコマンド | Read/Edit/Writeツール（対象外） |
| フック | 全ツール | 既知のAPIキーパターン、既知の危険コマンド | 未知のパターン、分割書き込み、エンコードされたキー |
| パーミッション | Read/Edit/Write/Bash | 機密ファイルの書き込み、破壊コマンド | パターンに一致しないファイル |
| ルール | Claudeの全行動 | AIの判断が必要なもの（設計、外部通信の妥当性） | Claudeが無視する可能性はゼロではない |

→ **サンドボックスはBashのみ、パーミッションはファイル操作のみ。単体では穴がある。4層を重ねることで相互補完する。**

## 検出対象一覧

### フックがブロックするAPIキー（content-secret-scan.sh）

| サービス | パターン |
|---|---|
| AWS | `AKIA...` / `aws_secret_access_key` |
| GitHub | `ghp_` / `gho_` / `ghs_` / `ghr_` / `github_pat_` |
| Slack | `xoxb-` / `xoxp-` / `xoxs-` / `xoxa-` |
| Google | `AIza...` / `AIzaSy...` |
| Anthropic | `sk-ant-...` |
| OpenAI | `sk-proj-...` / `sk-...`(40文字以上) |
| freee | freee + token パターン |
| npm | `_authToken` |
| Stripe | `sk_live_` / `pk_live_` / `sk_test_` |
| SendGrid | `SG.xxx.xxx` |
| Twilio | `AC` + 32文字hex |
| 秘密鍵 | `-----BEGIN PRIVATE KEY-----` |
| 汎用 | 変数名に key/secret/token/password/credential + リテラル値 |
| Authorization | Bearer ヘッダーへの直書き |

### フックが警告するコマンド（command-guard.sh）

| コマンド | リスク |
|---|---|
| `npm install` | postinstallスクリプト自動実行 |
| `pip install` | setup.py自動実行 |
| `npx` | typosquatting |
| `curl \| bash` | 任意コード実行 |
| `git push --force` | 履歴上書き |
| `git add .` / `git add -A` | 機密ファイル一括ステージング |
| `sudo` | 権限昇格 |
| `chmod 777` | パーミッション緩和 |
| `docker run` | コンテナ権限 |
| `curl $変数` | シークレット外部送信 |
| `curl -d @ファイル` | ファイル外部送信 |
| `rm -rf ~/` | ホームディレクトリ削除 |
| `eval $変数` | コードインジェクション |
| `base64 -d \| bash` | 難読化実行 |
| `git config --global` | グローバル設定変更 |
| `ngrok` | ローカルサーバー公開 |

## カスタマイズのヒント

- **検出パターンを追加**: `content-secret-scan.sh` に `echo "$CONTENT" | grep -qE 'パターン' && ISSUES="$ISSUES\n  説明"` を追加
- **警告コマンドを追加**: `command-guard.sh` に新しい `case` ブロックを追加
- **パーミッション拒否を追加**: `settings.json` の `deny` 配列にパターンを追加
- **プロジェクト固有ルール**: プロジェクトルートに `.claude/CLAUDE.md` を作成
- **settings.local.json の定期清掃**: 月1回、危険な allow パターンがないか確認する習慣をつける
