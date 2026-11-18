---
title: Magento DB 伺服器啟用 SSL 加密連線
date: 2026-11-18T17:00:00+08:00
draft: false
tags:
  - magento
  - mariadb
  - mysql
  - ssl
  - openssl
  - phpmyadmin
categories:
  - 自架網站
summary: APP 主機和 DB 主機拆開之後，Magento 連資料庫的流量要加密。這篇記錄在 DB 主機自建 CA 簽發憑證、MariaDB 啟用 SSL，再讓 phpMyAdmin 和 Magento env.php 都改走 SSL，最後強制 MySQL user 只能用 SSL 連線。
---

## 前言

Magento 的 APP 主機（運行 Magento 主程式）和 DB 主機分開之後，兩台之間的資料庫連線預設是明文的。這篇記錄把這段連線改成 SSL 加密的步驟，順序如下：
1. 在 DB 主機自建 CA，簽發伺服器憑證
2. MariaDB 啟用 SSL
3. APP 主機放置 `ca.pem`
4. phpMyAdmin 改用 SSL
5. Magento `env.php` 啟用 SSL 連線
6. MySQL user 強制只走 SSL

---

## 1. 在 DB 主機產生憑證

憑證效期設為 **3650 天（10 年）**。Magento 版本生命週期為 3 年，若每次升級時都重新建置主機，給憑證 3 年以上的效期就夠用。

```bash
sudo mkdir -p /etc/mysql/ssl && cd /etc/mysql/ssl
# 1. 自建 CA
sudo openssl genrsa -out ca-key.pem 4096
sudo openssl req -new -x509 -sha256 -days 3650 -key ca-key.pem -out ca.pem \
  -subj "/CN=Magento-DB-CA"
# 2. 伺服器金鑰與申請檔（CSR）
sudo openssl genrsa -out server-key.pem 3072
sudo openssl req -new -sha256 -key server-key.pem -out server-req.pem \
  -subj "/CN=magento-db"
# 3. SAN 填 DB 的私有 IP（APP 連線用的那個）
echo "subjectAltName=IP:<DB私有IP>" | sudo tee san.ext
# 4. CA 簽發伺服器憑證
sudo openssl x509 -req -sha256 -days 3650 -in server-req.pem \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -extfile san.ext -out server-cert.pem
# 5. 權限與檢查
sudo chown mysql:mysql *.pem
sudo chmod 600 *-key.pem
openssl x509 -in server-cert.pem -noout -dates -ext subjectAltName
```

| 檔案 | 用途 |
|---|---|
| `ca-key.pem` | CA 私鑰，只留在 DB 主機 |
| `ca.pem` | CA 憑證，要複製給 APP 主機驗證用 |
| `server-key.pem` | MariaDB 伺服器私鑰 |
| `server-cert.pem` | CA 簽發的伺服器憑證，SAN 是 DB 私有 IP |

> **⚠️ 注意：** SAN 一定要填 APP 實際連線用的 DB 私有 IP，否則開啟憑證驗證（`ssl_verify`）時會因為 IP 對不上而連線失敗。

---

## 2. MariaDB 啟用 SSL

在 `/etc/mysql/mariadb.conf.d/50-server.cnf` 的 `[mysqld]` 區段加入下列設定：

```ini
ssl_ca   = /etc/mysql/ssl/ca.pem
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key  = /etc/mysql/ssl/server-key.pem
```

重啟後確認 `have_ssl` 是 `YES`：

```bash
sudo systemctl restart mariadb
sudo systemctl status mariadb
sudo mysql -e "SHOW VARIABLES LIKE 'have_ssl';"
```

---

## 3. APP 主機放置 ca.pem

1. 將 `ca.pem` 複製到 APP 主機（本篇放在 `/etc/nginx/ssl/ca.pem`），權限設為 `644`。
2. 確認 PHP-FPM 的使用者讀得到，並比對兩台的指紋一致。

```bash
sudo chmod 644 /etc/nginx/ssl/ca.pem
# 兩台各跑一次，指紋要一樣
openssl x509 -in ca.pem -noout -fingerprint -sha256
```

---

## 4. phpMyAdmin 改用 SSL

設定寫在 `/etc/phpmyadmin/config.inc.php`，且要指定第 1 組伺服器 `[1]`：

```php
$cfg['Servers'][1]['host']       = '<DB私有IP>';
$cfg['Servers'][1]['ssl']        = true;
$cfg['Servers'][1]['ssl_ca']     = '/etc/nginx/ssl/ca.pem';
$cfg['Servers'][1]['ssl_verify'] = true;
```

---

## 5. Magento env.php 啟用 SSL 連線到 DB 主機

兩種做法擇一即可。

### 5.1 Option 1：直接修改 env.php

修改 `app/etc/env.php`，**改前先備份**：

```php
    'db' => [
        'table_prefix' => '',
        'connection' => [
            'default' => [
                'driver_options' => [
                    1014 => true,
                    1008 => '/etc/nginx/ssl/ca.pem'
                ]
            ]
        ]
    ],
```

### 5.2 Option 2：透過 Magento 指令調整 env.php

```bash
php bin/magento setup:config:set --db-ssl-ca=<路徑> --db-ssl-verify
# 重新啟動 PHP service
sudo systemctl restart php8.5-fpm.service
php bin/magento cache:flush
```

---

## 6. MySQL user 強制只走 SSL

**phpMyAdmin → 使用者帳號 → 選擇欲調整的 user → SSL → REQUIRE SSL**

> **💡 小提醒：** 這一步要放在最後。APP 和 phpMyAdmin 都確認能用 SSL 連線之後再設 REQUIRE SSL，否則還沒改好的連線會直接被拒絕。

---

## 小結：自建 CA 簽憑證，兩端都驗證，最後強制 SSL

- **DB 主機**：自建 CA 簽發伺服器憑證（SAN 填私有 IP），MariaDB 設定 `ssl_ca` / `ssl_cert` / `ssl_key`，`have_ssl = YES`
- **APP 主機**：放置 `ca.pem`，phpMyAdmin 和 Magento `env.php` 都開啟 SSL 與憑證驗證
- **收尾**：MySQL user 設定 REQUIRE SSL，確保不會再有明文連線
