---
title: 'WSL2上のNext.jsアプリにLAN内スマホからHTTPSでアクセスする方法'
emoji: '📱'
type: 'tech'
topics: ['wsl2', 'letsencrypt']
published: true
---

## はじめに

Next.jsなどのWebアプリケーションを開発する際、通常は開発サーバを用いてPC上のブラウザでスマートフォン用の表示をリアルタイムで確認します。
しかし、ミニアプリなどネイティブアプリ内のWebViewで表示される前提の機能の場合、PC上では再現できずネイティブアプリのインストールされたスマートフォン実機で確認する必要があります。

ただこの場合、以下の課題があります：

1. スマートフォンからPC上のローカル開発環境にアクセスしづらい
2. 最近のWebアプリの多くはHTTPSを要求する
3. スマートフォンでIPアドレスとポート番号を入力するのは煩雑

そこでこの記事では、**WSL2上で動作するNext.jsアプリに対して、LAN内のスマートフォンからHTTPSで簡単にアクセスする方法**を解説します。

## 本記事で実現する環境

```
スマートフォン(Android/iOS)
│ URL: https://dev.yourapp.your-domain.com/
│
▼ LAN内通信
Windows PC (192.168.x.x)
├ DNS: ドメインプロバイダ (dev.yourapp.your-domain.com → 192.168.x.x)
├ Docker Desktop
│ ├ CoreDNS（DNSサーバ）
│ └ nginx-proxy（HTTPS:443 → localhost:3000に転送）
│     └ Let's Encrypt SSL証明書
│
├ netsh portproxy: 3000番ポートをWSL2へ転送
▼
WSL2 Ubuntu
└ Next.jsアプリ(localhost:3000)
```

## 事前準備

- あなたが管理権限を持つドメイン（例: `your-domain.com`）
- 開発用サブドメイン（例: `dev.yourapp.your-domain.com`）
- LAN内固定IP（例: `192.168.x.x`）を設定したWindows PC
- Docker Desktopインストール済み
- WSL2環境セットアップ済み
- Next.jsプロジェクト ※他のフレームワークでも問題ないと思います

## (ちなみに)検討したが採用しなかった方法

### 1. 自己署名証明書を使用する方法

- 簡単に試せるが、スマートフォンのブラウザやWebViewではSSLエラーが発生
- 特にアプリ内WebViewでは自己署名証明書を信頼させるのが難しい

### 2. DNSを端末側で手動設定する方法

- PC側と端末側の両方で設定が必要で煩雑
- PC停止時に端末側のDNS設定を元に戻す必要があり運用が面倒

## 環境構築の手順

### 1. DNSレコード設定

所有しているドメインの管理画面（VercelやCloudflareなど）で以下のDNSレコードを追加します：

- Type: A
- Name: dev.yourapp（サブドメイン部分）
- Value: 192.168.x.x（あなたのPC固定IPアドレス）
- TTL: 3600（または推奨値）

### 2. SSL証明書取得

WSL2環境でCertbotを使用して証明書を取得します：

```bash
# Certbotインストール
sudo apt update
sudo apt install certbot

# 証明書取得（DNSチャレンジ方式）
sudo certbot certonly --manual --preferred-challenges dns -d dev.yourapp.your-domain.com
```

表示される指示に従い、一時的に以下のTXTレコードを追加：

- Type: TXT
- Name: \_acme-challenge.dev.yourapp
- Value: (Certbotが表示する値)

証明書取得後、以下の場所に証明書が保存されます：

- `/etc/letsencrypt/live/dev.yourapp.your-domain.com/fullchain.pem`
- `/etc/letsencrypt/live/dev.yourapp.your-domain.com/privkey.pem`

### 3. 証明書ファイルのコピー

これらのファイルをWindows環境にコピーします：

```bash
# Windowsの適切なディレクトリにコピー（WSL2から実行）
sudo cp /etc/letsencrypt/live/dev.yourapp.your-domain.com/fullchain.pem /mnt/c/path/to/your/docker-config/
sudo cp /etc/letsencrypt/live/dev.yourapp.your-domain.com/privkey.pem /mnt/c/path/to/your/docker-config/
sudo chmod 644 /mnt/c/path/to/your/docker-config/*.pem
```

### 4. Docker設定ファイル準備

`C:\path\to\your\docker-config`ディレクトリに以下のファイルを作成します：

**Corefile**:

```
. {
    hosts {
        192.168.x.x dev.yourapp.your-domain.com
        fallthrough
    }
    forward . 8.8.8.8 8.8.4.4
    log
}
```

**docker-compose.yml**:

```yaml
version: '3'

services:
  coredns:
    image: coredns/coredns:latest
    restart: always
    ports:
      - '53:53/udp'
      - '53:53/tcp'
    volumes:
      - ./Corefile:/Corefile

  nginx-proxy:
    image: nginx:latest
    restart: always
    ports:
      - '443:443'
    volumes:
      - ./default.conf:/etc/nginx/conf.d/default.conf
      - ./fullchain.pem:/etc/nginx/fullchain.pem
      - ./privkey.pem:/etc/nginx/privkey.pem
```

**default.conf**:

```nginx
server {
    listen 443 ssl;
    server_name dev.yourapp.your-domain.com;

    ssl_certificate /etc/nginx/fullchain.pem;
    ssl_certificate_key /etc/nginx/privkey.pem;

    location / {
        proxy_pass http://host.docker.internal:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 5. WSL2へのポート転送設定

WSL2のIPアドレスを確認します：

```bash
hostname -I
# 例: 172.16.x.x
```

Windows PowerShell（管理者権限）で以下を実行します：

```powershell
netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=172.16.x.x
```

ポート転送設定を確認：

```powershell
netsh interface portproxy show all
```

### 6. Docker起動

PowerShellで以下を実行します：

```powershell
cd C:\path\to\your\docker-config
docker compose down
docker compose up -d
```

コンテナが正しく起動していることを確認：

```powershell
docker ps
```

### 7. Next.jsアプリ起動

WSL2のUbuntuターミナルでNext.jsアプリを起動します：

```bash
cd ~/path/to/nextjs-app
npm run dev
# または
yarn dev
```

### 8. スマートフォンからアクセス

スマートフォンをPCと同じWi-Fiネットワークに接続し、ブラウザで以下のURLにアクセスします：

```
https://dev.yourapp.your-domain.com
```

## ワンクリック環境設定（Windows再起動後用）

ホストPC(Windows)を再起動すると、WSL2のIPアドレスが変わるため環境設定が必要です。
この作業を自動化するスクリプトとショートカットを用意することで、ワンクリックで環境設定が可能になります。

### 1. PowerShellスクリプトの作成

任意のディレクトリ（例: `C:\Scripts\dev-environment`）に以下のスクリプトを`start-dev-env.ps1`という名前で作成します：

```powershell
# Admin rights check
if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Warning "Administrator rights required."
    Start-Sleep -Seconds 2
    exit
}

# Check WSL2
Write-Host "Checking WSL2" -ForegroundColor Cyan
wsl -d Ubuntu -- echo "WSL2 Ubuntu running"

# Get WSL2 IP
$WSL_IP = (wsl -d Ubuntu hostname -I).Trim()
Write-Host "WSL2 IP: $WSL_IP" -ForegroundColor Green

# Update port forwarding
Write-Host "Clearing port forwarding" -ForegroundColor Cyan
netsh interface portproxy delete v4tov4 listenport=3000 listenaddress=0.0.0.0

Write-Host "Setting up port 3000 forwarding" -ForegroundColor Cyan
netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=$WSL_IP

# Check current port forwarding
Write-Host "Port forwarding setup:" -ForegroundColor Cyan
netsh interface portproxy show all

# Save current location
$currentLocation = Get-Location
Write-Host "Current location: $currentLocation" -ForegroundColor Cyan

# Restart Docker services
$dockerDir = "C:\path\to\your\docker-config"
if (Test-Path $dockerDir) {
    Write-Host "Restarting Docker" -ForegroundColor Cyan
    Set-Location $dockerDir

    # Error handling
    try {
        docker compose down
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "docker compose down failed (code: $LASTEXITCODE)"
        }

        docker compose up -d
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "docker compose up -d failed (code: $LASTEXITCODE)"
        }

        Write-Host "Docker restart complete" -ForegroundColor Green
    }
    catch {
        Write-Error "Error during Docker operation: $_"
    }
    finally {
        # Return to original location
        Set-Location $currentLocation
    }
}
else {
    Write-Warning "Docker config directory not found: $dockerDir"
}

Write-Host "Environment setup complete" -ForegroundColor Green
Write-Host "URL: https://dev.yourapp.your-domain.com/" -ForegroundColor Yellow
Write-Host "Next.js startup: Run 'npm run dev' or 'yarn dev' in WSL2" -ForegroundColor Yellow
Start-Sleep -Seconds 2
```

### 2. デスクトップショートカットの作成

1. デスクトップの何もない場所で右クリック → 「新規作成」→「ショートカット」を選択
2. 以下のコマンドを入力（パスは適宜修正）:

   ```
   powershell -ExecutionPolicy Bypass -Command "Start-Process powershell -ArgumentList '-ExecutionPolicy Bypass -File \"C:\Scripts\dev-environment\start-dev-env.ps1\"' -Verb RunAs"
   ```

3. ショートカット名を「Development Environment Setup」などのわかりやすい名前に設定
4. ショートカットのアイコンを右クリック→「プロパティ」→「詳細設定」→「管理者として実行」にチェック

### 3. スクリプト作成時の注意点

- PowerShellスクリプトファイルは「UTF-8 with BOM」形式で保存する
- VSCodeを使用する場合、保存時にエンコーディングを「UTF-8 with BOM」に設定
- `$dockerDir`の値はあなたの環境に合わせて変更する
- スクリプト内のドメイン名（`dev.yourapp.your-domain.com`）は実際のドメインに置き換える

### 4. 使用方法

1. Windows再起動後にデスクトップショートカットをダブルクリック
2. 管理者権限の確認ダイアログで「はい」をクリック
3. WSL2でNext.jsアプリを起動:

   ```bash
   cd ~/path/to/nextjs-app
   npm run dev  # または yarn dev
   ```

4. スマートフォンブラウザで <https://dev.yourapp.your-domain.com/> にアクセス

## 運用上の注意点

### SSL証明書の更新

Let's Encrypt証明書は90日間有効です。更新するには：

```bash
sudo certbot renew
```

証明書が更新されたら、Windows環境にコピーし直す必要があります。
この記事では自動更新には対応していません。

### WSL2のIPアドレス変更時の対応

WSL2は再起動するとIPアドレスが変わります。前述のワンクリック環境設定スクリプトを使うと自動的に処理されますが、手動で行う場合は以下の手順で更新します：

1. WSL2の新しいIPアドレスを確認

   ```bash
   hostname -I
   ```

2. ポート転送設定を更新（PowerShell管理者権限で）

   ```powershell
   netsh interface portproxy delete v4tov4 listenport=3000 listenaddress=0.0.0.0
   netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=新しいIPアドレス
   ```

### ファイアウォール設定

Windows Defenderファイアウォールで以下のポートを許可します：

- TCP 443（HTTPS）
- TCP/UDP 53（DNS）
- TCP 3000（Next.js）

## トラブルシューティング

### スマホからアクセスできない場合

1. DNSレコードが正しく設定されているか確認
2. Dockerコンテナの稼働状況を確認（`docker ps`）
3. ポート転送設定を確認（`netsh interface portproxy show all`）
4. スマホのDNSキャッシュをクリア（機内モードをオン/オフ）

### SSL証明書エラーが表示される場合

1. 証明書ファイルが正しくコピーされているか確認
2. nginx設定が正しいか確認
3. 証明書の有効期限を確認

### Next.jsアプリが表示されない場合

1. WSL2でNext.jsアプリが起動しているか確認
2. WSL2のIPアドレスが変わっていないか確認
3. ポート3000が他のアプリで使用されていないか確認

## まとめ

この方法を使えば、実際に所有しているドメインとLet's Encryptの証明書を活用して、LAN内のスマートフォンからWSL2上のNext.jsアプリにHTTPSで簡単にアクセスできます。特にミニアプリなど、実機での検証が重要な開発において、この環境構築は非常に役立つでしょう。

一度設定してしまえば、ワンクリック環境設定スクリプトを活用することで、毎回の開発環境立ち上げを簡略化できます。SSL証明書の更新やWSL2のIP変更にも対応できるため、長期間の開発も快適に進められます。

### 自分の環境用にカスタマイズするための置換リスト

最後にこの記事の手順やコードをあなたの環境で使用するための、一斉置換できる項目一覧を置いておきます。

| 置換対象                        | 説明                                       | 置換例                            |
| ------------------------------- | ------------------------------------------ | --------------------------------- |
| `dev.yourapp.your-domain.com`   | あなたの所有するドメインとサブドメイン     | `dev.myapp.example.com`           |
| `192.168.x.x`                   | Windows PCのLAN内固定IPアドレス            | `192.168.1.50`                    |
| `172.16.x.x`                    | WSL2のIPアドレス（実行時に変わる）         | `172.24.53.12`                    |
| `C:\path\to\your\docker-config` | Docker設定ファイルを格納するディレクトリ   | `C:\Users\username\docker-config` |
| `~/path/to/nextjs-app`          | Next.jsアプリケーションのWSL2内パス        | `~/projects/my-nextjs-app`        |
| `C:\Scripts\dev-environment`    | PowerShellスクリプトを格納するディレクトリ | `C:\Users\username\Scripts`       |
