# Claude Code 安全運用ガイドライン

> このファイルはClaude Codeが自動的に読み込む設定ファイルです。
> チーム全員がレビューし、プロジェクトルートに配置してください。

## 🚫 禁止コマンド（BANNED COMMANDS）

Claude Codeは、いかなる理由があっても以下のコマンドを**実行してはならない**。
ユーザーが明示的に指示した場合も、必ず確認を求めること。

### ファイル・ディレクトリ削除

```
# 絶対禁止
rm -rf
rm -f
rmdir /s /q        # Windows
del /f /q          # Windows

# 要確認（実行前にすべてユーザーに提示）
rm <file>
find . -delete
git clean -fdx
```

### 破壊的なディスク操作

```
dd if=...          # ディスク直接書き込み
mkfs.*             # フォーマット
fdisk / diskpart   # パーティション操作
shred / wipe       # 安全消去
truncate -s 0      # ファイル内容消去
> <file>           # リダイレクトによる上書き（意図がない場合）
```

### プロセス・システム操作

```
kill -9 -1         # 全プロセス強制終了
shutdown / reboot / halt
systemctl disable  # サービス無効化
crontab -r         # cron全削除
```

## 📦 パッケージ管理の安全規則

### npm（Node.js）

```
# 禁止・要注意
npm install <パッケージ名>          # 事前レビューなしの新規追加 → 禁止
npm install --ignore-scripts       # postinstallスクリプト無効化は推奨だが省略禁止
npm run <script>                   # package.jsonの内容を必ず確認してから実行

# 必須手順
# 1. パッケージ追加前にnpms.io / Socket.devでスコアを確認
# 2. package-lock.jsonをコミットに含める（.gitignore禁止）
# 3. npm audit を定期実行し、HIGH/CRITICAL は即時対応
```

**サプライチェーン攻撃 チェックリスト（npm）**

- [ ] パッケージ名のタイポスクワッティング確認（例: `lodash` vs `1odash`）
- [ ] 週間DL数・GitHubスター数・最終更新日を確認
- [ ] `package.json` の `scripts.postinstall` に不審なコマンドがないか確認
- [ ] 依存パッケージの推移的依存を `npm ls` で確認
- [ ] `npx <見知らぬコマンド>` は原則禁止（キャッシュなしでリモート実行される）

```
# 危険な例（禁止）
npx some-unknown-tool              # 公式確認なしのnpx実行
npm install $(cat user_input)      # 動的パッケージ名
```

### pip（Python）

```
# 禁止・要注意
pip install <パッケージ名>         # 事前レビューなしの新規追加 → 禁止
pip install --trusted-host ...    # SSL検証の無効化 → 禁止
pip install -r requirements.txt   # 固定バージョン（==）なしは禁止

# 必須手順
# 1. requirements.txtは必ず == でバージョン固定
# 2. pip-audit または safety check を定期実行
# 3. 仮想環境（venv / poetry）の外へのグローバルインストール禁止
```

**サプライチェーン攻撃 チェックリスト（pip）**

- [ ] PyPI上のパッケージ名と公式ドキュメントのパッケージ名が一致するか確認
- [ ] `setup.py` / `pyproject.toml` に不審なスクリプトがないか確認
- [ ] `pip install` 前に `pip index versions <pkg>` でバージョン存在確認
- [ ] ハッシュ検証を活用（`pip install --require-hashes -r requirements.txt`）

## 🔑 シークレット・認証情報の取り扱い

### 絶対禁止

```
# コードやチャットへの直接記載
API_KEY="sk-xxxxxxxxxxxx"          # ソースコードへのハードコード禁止
password = "my_password"           # 同上

# 履歴に残る形での使用
export SECRET=xxxxx && ...         # シェル履歴に残る
echo $SECRET | some-command        # ログに残る可能性
curl -H "Authorization: Bearer hardcoded-token"
```

### 必須ルール

1. **シークレットは `.env` ファイルに記載し、`.gitignore` に追加済みであることを確認してから使用**
2. Claude Codeは、いかなる場合も `.env` の内容をチャットに出力しない
3. `git log` や `git diff` の出力にシークレットが含まれていないか確認してからPush
4. シークレットスキャナーツールの導入を推奨（`gitleaks`、`truffleHog`）

```
# .gitignoreに必ず含めること
.env
.env.*
*.pem
*.key
*_secret*
*credentials*
```

> Claude Codeは環境変数やシークレットの値を質問されても、**チャット上に表示してはならない**。
> 代わりに、`echo $VARIABLE_NAME` で確認してもらうよう案内すること。

## 🌿 Git操作の安全規則

### 禁止コマンド

```
# 履歴書き換え系（チームリポジトリでは禁止）
git push --force                   # 禁止 → --force-with-lease を使うこと
git rebase <共有ブランチ>           # main/develop への直接rebaseは禁止
git reset --hard HEAD~n            # 複数コミット遡行は要確認
git filter-branch                  # 履歴改変 → 禁止（git-filter-repo を使うこと）

# 危険な操作
git clean -fdx                     # 未追跡ファイル全削除 → 禁止
git checkout -- .                  # ワーキングツリー全リセット → 要確認
git stash drop                     # スタッシュ削除 → 確認必須
```

### 必須ルール

1. **`main` / `develop` への直接pushは禁止。必ずPull Requestを経由する**
2. Claude CodeがcommitするときはI、変更ファイル一覧をユーザーに提示してから実行
3. `git commit --amend` は未Pushのコミットのみ許可
4. タグの削除（`git tag -d` / `git push --delete origin <tag>`）は要確認

### コミット前チェック

```bash
# Claude Codeはコミット前に以下を確認すること
git diff --stat          # 変更ファイルの確認
git status               # 意図しないファイルが含まれていない
git log --oneline -5     # 直近の履歴の確認
```

## 🖥️ VSCode連携の注意点

### 拡張機能

- **新規拡張機能のインストールをClaude Codeが提案する場合、拡張機能IDを明示すること**（例: `ms-python.python`）
- Marketplace上の類似名の偽拡張機能に注意（公式発行者IDを確認）
- 拡張機能の設定（`settings.json`）にシークレットを記載しない

### `.vscode/` ディレクトリ

```
# チームで共有していいもの
.vscode/settings.json       # エディタ設定（シークレット不可）
.vscode/extensions.json     # 推奨拡張機能リスト
.vscode/launch.json         # デバッグ設定（認証情報不可）

# .gitignoreに追加すること
.vscode/*.log
```

タスク・デバッグ設定の `tasks.json` / `launch.json` にシェルコマンドを記載する場合、Claude Codeはその内容をユーザーに提示してから承認を得ること。

## ✅ Claude Codeへの行動規範

| 状況 | Claude Codeの対応 |
|------|-----------------|
| 破壊的コマンドを求められた | 内容・影響範囲を提示し**明示的な確認後**に実行 |
| 新規パッケージ追加 | パッケージ名・バージョン・目的を提示してから追加 |
| `.env` の内容を聞かれた | チャットに出力せず、確認方法を案内 |
| `main`へ直接push | 拒否し、PRフローを案内 |
| `rm -rf` を含むコマンド | 必ず中断し、代替手段を提案 |
| 不審なCLIツールの実行 | 公式リポジトリのURLを確認してからのみ実行 |

## 🔄 定期メンテナンス

```bash
# 週次推奨
npm audit                          # 脆弱性チェック
pip-audit                          # Python脆弱性チェック
gitleaks detect                    # シークレット漏洩スキャン

# 月次推奨
npm outdated                       # 依存パッケージの更新確認
pip list --outdated
```

---

*最終更新: プロジェクト開始時に日付を記入してください*
*レビュー担当: チームリーダー*
