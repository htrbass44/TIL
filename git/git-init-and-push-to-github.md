# Git入門 - リポジトリの作成とGitHubへのPush

## 学んだこと

ローカルにGitリポジトリを作成し、GitHub上のリモートリポジトリと連携してPushするまでの一連の流れを学んだ。

## コマンド

### 1. リポジトリの初期化

```bash
git init
```

カレントディレクトリをGitリポジトリとして初期化する。

### 2. README.mdの作成とコミット

```bash
echo "# TIL (Today I Learned)" > README.md
git add README.md
git commit -m "first commit"
```

- `git add`: 変更をステージングエリアに追加する
- `git commit -m "メッセージ"`: ステージされた変更をコミットする

### 3. メインブランチの名前をmainに設定

```bash
git branch -M main
```

デフォルトブランチ名を`main`に変更（リネーム）する。

### 4. リモートリポジトリの登録

```bash
git remote add origin https://github.com/htrbass44/TIL.git
```

GitHub上の空リポジトリを`origin`という名前でリモートに登録する。

### 5. GitHubへPush

```bash
git push -u origin main
```

ローカルの`main`ブランチをリモート`origin`にPushする。`-u`オプションを付けることで、以後は`git push`だけでこのリモートブランチに追跡・Pushできるようになる。

### 6. 新しいフォルダ・ファイルの追加をPush

```bash
git add git/
git commit -m "add git folder"
git push
```

新しく作成したフォルダ（例: `git`フォルダ）やファイルをステージング・コミットし、リモートにPushする。初回に`git push -u origin main`で追跡設定済みのため、2回目以降は`git push`のみでよい。

## メモ

- `origin`はリモートリポジトリのデフォルトの呼び名（別名でもよい）
- `-u`（`--set-upstream`）は最初の1回だけ付ければ、次回以降は追跡設定が省略できる
