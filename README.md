# ApacheGuacamole-Setup

Apache Guacamole を利用したリモートアクセス

---

## 経緯

1. いつものように[オープンソースカンファレンス](https://www.ospn.jp/osc2019-fall/)に参加したところ、**Apache Guacamole**の紹介セミナーを発見
1. 社内で実施しているモブプログラミング形式の勉強会で使えそう
1. 早速 Azure 上の VM(Windows Server 2019)に WSL を用意したうえで、Guacamole をインストールしてみた

---

## Apache Guacamoleとは

読み方: アパッチ ワカモレ

```
Apache Guacamoleは、クライアントレス(プラグインやクライアントソフトウェアが要らない)リモートデスクトップゲートウェイで、VNC、RDP、SSHのような標準的なプロトコルに対応しています。HTML5のおかげで、いったんサーバーにGuacamoleをインストールしてしまえば、デスクトップにアクセスするために必要なのはWebブラウザだけです。

https://guacamole.apache.org/
```

### 主な特徴

* クライアント側にWebブラウザだけしか必要ない。アドオンすら入れる必要がないため、ソフトウェアのインストールに制約がある(社内)環境でも導入しやすく、詳しくないユーザーを招待しやすい
* 接続可能なホストを、ユーザーやグループごとに設定できる
* 入力されたコマンド(CUI)や、画面キャプチャの動画(GUI)を保存できるので、証跡として利用したり監査を行ったりできる
* 1つのホストの画面を複数人で共有できる。閲覧専用にすることも、操作させることも可能

* クライアント側は導入する手間がほとんどかからないが、サーバー側のセットアップが若干面倒

## 構成

クライアント PC(Windows, Linux, Mac)上のモダンブラウザ → インターネット → Azure 仮想ネットワーク・NSG → Azure VM(Windows Server 2019) → WSL(Ubuntu 18.04) → Guacamole

## インストール手順

### MySQL

```sh
$ sudo apt update -y && sudo apt upgrade -y && sudo apt install mysql-server mysql-client -y

$ sudo service mysql start # 起動しておかないと初期化の際にエラーが発生する
$ sudo rm -rf /var/lib/mysql && sudo mysqld --initialize
$ grep 'temporary password' /var/log/mysql/error.log
```

初期仮パスワードを確認

```
2019-11-23T12:22:56.144754Z 1 [Note] A temporary password is generated for root@localhost: ************
```

初期パスワードから任意のパスワードに変更

```sh
$ sudo service mysql stop && sudo usermod -d /var/lib/mysql mysql && sudo service mysql start # 起動しておかないと初期設定の際にエラーが発生する
$ sudo mysql_secure_installation
```

ログインできることを確認

```sh
$ sudo mysql -u root -p
```

### Guacamole

[インストールスクリプト](https://github.com/MysticRyuujin/guac-install)を利用する。
以下のコマンドのまま実行すると二要素認証(TOTP)が有効化されるので、不要であればコマンドオプション(`-n` または `--nototp`)をつけるか、スクリプトを変更する(`installTOTP=false`)

```sh
$ wget https://git.io/fxZq5 && mv fxZq5 guac-install.sh && chmod +x guac-install.sh
$ sudo ./guac-install.sh --mysqlpwd *** --guacpwd ***
```

### Windows Server(リモートデスクトップ先のホスト)

RDPで接続するために、レジストリを変更する [†](https://blog.happynavy.tk/guacamole-rdp-windows-10-and-windows-server-2016/)

```
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp]
"SecurityLayer"=dword:00000001
"UserAuthentication"=dword:0x00000000
```

### ブラウザでのセットアップ

- Windows Server 2019のブラウザから[アクセス](http://localhost:8080/guancamole/)して、guacadmin のパスワードを変更

- localhost 接続情報を作成

![接続情報](src/01-001.png "接続情報")

```
Protocol: RDP
Hostname: localhost
Port: 3389
Username: ***
Password: ***
```

読み書き / 読み取り専用のプロファイルを作成

![プロファイル](src/01-002.png "プロファイル")

権限弱めのユーザーを作成

![ユーザー作成](src/01-003.png "ユーザー作成")

(Azure NSG のポート開放と、)ファイアウォールの設定変更

![ファイアウォール](src/02-002.png "ファイアウォール") ![ファイアウォール](src/02-003.png "ファイアウォール")

クライアント PC から、[http://<domain>:8080/guancamole/](http://<domain>:8080/guancamole/)にアクセスし、サインインする

リモートデスクトップ接続の画面を表示している状態で`Ctrl+Alt+Shift`を押すと、メニューが表示される

![Ctrl+Alt+Shift](src/03-001.png "Ctrl+Alt+Shift")

必要に応じて、読み書き / 読み取り専用のプロファイルどちらかを選択し URL を発行、その URL を誰かに連携

![共有](src/03-002.png "共有")

![共有(Firefox)](src/03-003.png "共有(Firefox)") ![共有(Chrome)](src/03-004.png "共有(Chrome)") ![共有(Edge)](src/03-005.png "共有(Edge)")

---

Copyright (c) 2019 YA-androidapp(https://github.com/YA-androidapp) All rights reserved.
