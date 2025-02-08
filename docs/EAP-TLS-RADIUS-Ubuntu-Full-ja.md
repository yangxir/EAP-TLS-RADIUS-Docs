# Ubuntu における EAP-TLS RADIUS サーバー

本プロジェクトでは、Ubuntu 環境で **EAP-TLS 認証を使用した FreeRADIUS サーバー** のセットアップ手順を提供します。以下の内容を含みます：
- **FreeRADIUS のインストール**
- **EAP-TLS 証明書の生成**
- **RADIUS サーバーの設定**
- **クライアントデバイスの接続**
- **デバッグとトラブルシューティング**

---

## 🚀 実験環境
1. **Ubuntu 仮想マシン**（RADIUS サーバーとして）。
2. **ルーター**（無線アクセスポイントとして）。
3. **Redmi K50 スマートフォン**（Wi-Fi クライアントとして）。
4. **無線アダプター**（デバッグやパケットキャプチャ用）。

---

## 📌 ディレクトリ構成
```
/EAP-TLS-RADIUS-Ubuntu
│── README.md
│── docs/
│   ├── setup-guide.md  # Ubuntu の詳細なセットアップ手順
│   ├── troubleshooting.md  # 一般的なエラーとその解決策
│── configs/
│   ├── freeradius/
│   │   ├── eap.conf  # FreeRADIUS のメイン設定
│   │   ├── clients.conf  # 許可された RADIUS クライアント
│   │   ├── mods-enabled/
│   │   ├── certs/  # 証明書関連ファイル
│── scripts/
│   ├── install_freeradius.sh  # FreeRADIUS のワンクリックインストール
│   ├── generate_certs.sh  # 証明書生成スクリプト
│   ├── start_radius.sh  # FreeRADIUS の起動
│   ├── test_auth.sh  # radtest での認証テスト
│── examples/
│   ├── ubuntu_client.md  # Ubuntu デバイスの接続方法
│   ├── windows_client.md  # Windows デバイスの接続方法
│── LICENSE
│── .gitignore
```

---

## 📌 1. FreeRADIUS のインストール
```sh
sudo apt update && sudo apt install freeradius freeradius-utils -y
```

RADIUS サーバーが正常に動作しているか確認：
```sh
sudo systemctl status freeradius
```

起動していない場合：
```sh
sudo systemctl enable --now freeradius
```

---

## 📌 2. FreeRADIUS の設定

### **`clients.conf` の編集**
```sh
sudo nano /etc/freeradius/3.0/clients.conf
```
次の内容を追加してください（⚠️ **`192.168.51.2` をルーターの IP に置き換えてください** ⚠️）：
```ini
client your_router {
    ipaddr = 192.168.51.2  
    secret = testing123  
    shortname = router  
}
```

### **EAP モジュールの有効化**
```sh
sudo nano /etc/freeradius/3.0/mods-enabled/eap
```
次の設定を確認してください：
```ini
eap {
    default_eap_type = tls
}

tls-config tls-common {
    private_key_file = /etc/freeradius/3.0/certs/server.key
    certificate_file = /etc/freeradius/3.0/certs/server.crt
    CA_file = /etc/freeradius/3.0/certs/ca.crt
    dh_file = /etc/freeradius/3.0/certs/dh
    random_file = /dev/urandom
}
```

⚠️ **私の証明書のパスを使用していますので、必ず自分の証明書のパスに置き換えてください！**

---

## 📌 3. 証明書の生成
証明書ディレクトリを作成（存在しない場合）：
```sh
sudo mkdir -p /etc/freeradius/3.0/certs
cd /etc/freeradius/3.0/certs
```

### **CA 証明書の生成**
```sh
sudo openssl genrsa -out ca.key 2048
sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=CA"
```

### **サーバー証明書の生成**
```sh
sudo openssl genrsa -out server.key 2048
sudo openssl req -new -key server.key -out server.csr -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=radius-server.local"
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256
```

### **クライアント証明書の生成**
```sh
sudo openssl genrsa -out client.key 2048
sudo openssl req -new -key client.key -out client.csr -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=client"
sudo openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256
```

⚠️ ***注意：
上記のコマンドにある "Tohoku University" や "Goto Lab" などは私の研究室の情報ですので、ご自身の組織名に変更してください！***

### **PKCS#12 クライアント証明書の作成（スマートフォン用）**
```sh
sudo openssl pkcs12 -export -inkey client.key -in client.crt -certfile ca.crt -out client.p12
```

---

## 📌 4. FreeRADIUS の起動とテスト
```sh
sudo systemctl restart freeradius
sudo journalctl -u freeradius -f
```

`radtest` を使用して認証テスト：
```sh
radtest user password 127.0.0.1 0 testing123
```
成功すると、`Access-Accept` が返されます。

---

## 📌 5. クライアントの Wi-Fi 設定

### **クライアント証明書の準備**
1. `client.p12` と `ca.crt` をスマートフォンに転送します（Google Drive などを使用）。
2. Android/iPhone に証明書をインポート：
   - **Wi-Fi の詳細設定** で **EAP-TLS 認証** を選択。
   - `client.p12` をクライアント証明書として選択。
   - `ca.crt` を CA 証明書として選択。

---

## 📜 ライセンス
本プロジェクトは **MIT License** の下で提供されています。自由に使用・変更できますが、引用または二次開発する場合は出典を明記してください。

---

## 📚 参考資料
本プロジェクトは以下のリソースに基づいています：
- **FreeRADIUS 公式ウェブサイト**: [https://www.freeradius.org/](https://www.freeradius.org/)
- **NII 学術認証推進室 - Eduroam & RADIUS 関連ドキュメント**: [NII MeatWiki](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=94340973)
  
---

## 📩 連絡先
質問や提案がある場合は、Issue を提出するか、私の GitHub ページにアクセスしてください：

🔗 [GitHub プロフィール](https://github.com/yangxir)  

このプロジェクトが役に立った場合は **Star ⭐** をお願いします！
