
# Amazon EC2でRadius サーバを構築する

## 構築手順

### freeradius と google-authenticator の 設定

RADIUSサーバーに必要なパッケージのインストールです。 下記のコマンドを実行してインストールしましょう。

```bash
sudo su
sudo yum -y install freeradius freeradius-utils git gcc pam-devel qrencode qrencode-libs qrencode-devel autoconf automake libtool
cd
git clone https://github.com/google/google-authenticator-libpam.git
cd /root/google-authenticator-libpam/
./bootstrap.sh
./configure
make
sudo make install
```

RADIUS経由の認証を許可するグループを作成します。 RADIUSサーバーは個別にユーザーを作る必要があるのですが、この際にこのグループに所属させておきます。

```bash
sudo groupadd radius-enabled
```

### ホスト名の設定

ホスト名を設定しておきます。ホスト名は、後ほどGoogle Authenticatorに登録した際に、登録名で使われます。

```bash
sudo vi /etc/hosts
```

`192.168.0.124 radius.mfa.test` を記載する。

sudo hostnamectl set-hostname radius.mfa.test

### FreeRadius のデフォルト設定

```bash
sudo vi /etc/raddb/users
```

DEFAULT    Group != "radius-enabled", Auth-Type := Reject
           Reply-Message = "Your account has been disabled."
DEFAULT    Auth-Type := PAM

```bash
sudo vi /etc/raddb/radiusd.conf
```

user = root
group = root

### radius の PAMを有効化

```bash
sudo vi /etc/raddb/sites-available/default
```

```text
# Pluggable Authentication Modules.
pam
```

### RadiusでGoogoleAuthenticatorが使われるように設定

```bash
sudo vi /etc/pam.d/radiusd
```

```text
#%PAM-1.0
#auth       include     password-auth
#account    required    pam_nologin.so
#account    include     password-auth
#password   include     password-auth
#session    include     password-auth
auth requisite pam_google_authenticator.so
account required pam_permit.so
session required pam_permit.so
```

### RADIUSがプロトコルを受け付けるクライアントの設定

```bash
sudo vi /etc/raddb/clients.conf
```

```text
client vpc {
        ipaddr = 0.0.0.0/0
        secret = testing65536
}
```

### pam モジュールのシンボリックリンクを追加

```bash
sudo ln -s /etc/raddb/mods-available/pam /etc/raddb/mods-enabled/pam
sudo cp /usr/local/lib/security/pam_google_authenticator.so /usr/lib64/security
```

### Radius サーバを起動

```bash
sudo systemctl enable radiusd
sudo systemctl stop radiusd
sudo systemctl start radiusd
systemctl status radiusd.service
```

### RADIUSサーバーが認証するユーザーの設定

RADIUSサーバーはActive Directoryと連携している訳ではないので、RADIUSサーバー上にもActive Directoryと同じユーザーの追加が必要です。
Active Directory上ユーザー名を指定し、Linux上にユーザーを追加します。

```bash
useradd -g radius-enabled mfa_test
```

ユーザーを追加したら、追加したユーザーの権限で、google-authenticatorを起動しましょう。

```bash
sudo -u mfa_test /usr/local/bin/google-authenticator
```

/usr/sbin/radiusd -C -lstdout -xxx
