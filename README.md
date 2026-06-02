# Laravel Docker Template

Laravel開発用のDockerテンプレートです。
PHP / nginx / MySQL / phpMyAdmin / Node / MailHog を含みます。

## 使用技術
開発時の状況に適したバージョンへ適宜変更すること。
（スクール提出用の場合は、3桁目までバージョンを指定したほうが良い）

* PHP 8.3
* nginx 1.29-alpine
* MySQL 8.4.6
* phpMyAdmin 5.2.2
* Node 22.17.0
* MailHog
* Laravel 13

## ディレクトリ構成

```txt
project-root/
├── docker-compose.yml
├── docker/
│   ├── php/
│   │   ├── Dockerfile
│   │   └── php.ini
│   ├── nginx/
│   │   └── default.conf
│   └── mysql/
│       └── my.cnf
└── src/
    └── Laravel本体
```

ホスト側の `src` は、コンテナ内では `/var/www` にマウントされます。

```txt
ホスト側: ./src
コンテナ内: /var/www
```

そのため、Laravelの `artisan`、Composer、npm、Vite はすべて `/var/www` を基準に実行します。

## 初回セットアップ手順

### 1. コンテナを起動

プロジェクトルートで実行します。

```bash
docker compose up -d --build
```

起動確認。

```bash
docker ps
```

以下のコンテナが起動していればOKです。

```txt
php
nginx
mysql
node
phpmyadmin
mailhog
```

### 2. Laravelを作成

`src` が空の状態で、PHPコンテナに入ります。

```bash
docker compose exec php bash
```

コンテナ内で実行します。

```bash
composer create-project laravel/laravel .
```

`.` は「現在の `/var/www` にLaravelを作成する」という意味です。

### 3. .env を設定

`src/.env` を編集します。

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=attendance
DB_USERNAME=laravel_user
DB_PASSWORD=laravel_pass

MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="test@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

設定を反映します。

```bash
docker compose exec php php artisan config:clear
```

MySQLにテーブルを作成します。

```bash
docker compose exec php php artisan migrate:fresh
```

### 4. Viteを起動

Nodeコンテナで実行します。

```bash
docker compose exec node npm install
docker compose exec node npm run dev -- --host 0.0.0.0
```

Docker環境では、Viteを外部から参照できるように `--host 0.0.0.0` を付けます。

## Vite設定の注意点

Docker環境では、ブラウザ側に `http://0.0.0.0:5173` が出るとCSSやHMRが正常に動かないことがあります。

そのため、`src/vite.config.js` の `server` 設定は以下のようにします。

```js
server: {
    host: '0.0.0.0',
    port: 5173,
    strictPort: true,
    hmr: {
        host: 'localhost',
    },
    watch: {
        ignored: ['**/storage/framework/views/**'],
    },
},
```

ページソース（検証ページ）で以下のように表示されればOKです。

```html
http://localhost:5173/resources/css/app.css
```

## 権限設定の注意点

PHPコンテナは以下のように、UID/GIDを `1000` に揃えています。

```yaml
args:
  UID: 1000
  GID: 1000
user: "1000:1000"
```

これにより、WSL側のユーザーとPHPコンテナ内の `www-data` が同じIDとして扱われます。

```bash
docker compose exec php id
```

結果が以下ならOKです。

```txt
uid=1000(www-data) gid=1000(www-data)
```

Laravelが書き込む主なディレクトリは以下です。

```txt
src/storage
src/bootstrap/cache
```

権限確認。

```bash
ls -ld src/storage
ls -ld src/storage/framework
ls -ld src/storage/framework/views
ls -ld src/bootstrap/cache
```

必要に応じて修正します。

```bash
sudo chown -R $USER:$USER src/storage src/bootstrap/cache
chmod -R 775 src/storage src/bootstrap/cache
```

## ブラウザ確認

Laravel

```txt
http://localhost
```

phpMyAdmin

```txt
http://localhost:8080
```

MailHog

```txt
http://localhost:8025
```

## phpMyAdminログイン情報

```txt
サーバー: mysql
ユーザー名: laravel_user
パスワード: laravel_pass
```

## MySQL接続情報

```env
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=laravel_user
DB_PASSWORD=laravel_pass
```
DATABASE名はプロジェクトに応じて命名すること。

## MailHog設定

Laravelの `.env` では以下を使用します。

```env
MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
```

送信メールは以下で確認します。

```txt
http://localhost:8025
```

## Git管理しないもの

プロジェクトルートの `.gitignore` で、以下を除外します。

```gitignore
# Laravel dependencies
src/vendor
src/node_modules

# Laravel environment
src/.env
src/.env.backup
src/.env.production

# Laravel build/cache
src/public/build
src/public/hot
src/storage/*.key

# Logs
*.log

# Docker local data
docker/mysql/data

# Editor
.vscode
.idea

# OS
.DS_Store
Thumbs.db
```

`src/.env`、`src/vendor/`、`src/node_modules/` はGitに含めません。

確認コマンド。

```bash
git status --ignored
```

## よく使うコマンド

Laravelキャッシュ削除。

```bash
docker compose exec php php artisan optimize:clear
```

Vite起動。

```bash
docker compose exec node npm run dev -- --host 0.0.0.0
```

## 注意事項

* Laravel本体は必ず `src` 以下に置きます。
* コンテナ内では Laravel 本体は `/var/www` です。
* `composer` はPHPコンテナ内で実行します。
* `npm` はNodeコンテナ内で実行します。
* WSL側にPHP / Composer / Nodeを入れる必要はありません。
* `.env` はGitHubにpushしません。
* Viteの `public/hot` もGit管理しません。
* 権限エラーが出た場合は、まず `docker compose exec php id` と `ls -l` でUID/GIDを確認します。

## 初回コミットの目安

以下が確認できたら、環境構築完了としてコミットします。

* Laravelが `http://localhost` で表示できる
* phpMyAdminが `http://localhost:8080` で表示できる
* MailHogが `http://localhost:8025` で表示できる
* MySQLに `migrate:fresh` できる
* ViteでCSS変更が反映される
* `src/.env` がGit管理対象外になっている

```bash
git add .
git commit -m "環境構築完了"
```
