# FreeBSD/jail環境でのHTTP/2接続問題のトラブルシューティング

## 概要

FreeBSD上でjail環境を使用したWebサーバ構成において、HTTP/2プロトコルでの接続が正常に動作しない問題が発生しました。本記事では、その問題の詳細と考察を記録します。

## 環境構成

### システム構成
- **OS**: FreeBSD
- **接続**: NTTフレッツ経由でのppp-over-ip tunnel
- **ネットワーク**: 公開IPアドレス使用
- **仮想化**: FreeBSD jail
- **プロキシ**: nginx（リバースプロキシ、TLS/SNI対応）
- **Webサーバ**: nginx（WordPressコンテンツサーバ）
- **ファイアウォール**: PF（Packet Filter）

### ネットワーク設定
```
インターネット → NTTフレッツ → PPP-over-IP tunnel → FreeBSD host
                                                    ↓ PF (443/tcp redirect)
                                                    jail: nginx reverse proxy
                                                    ↓ 内部転送
                                                    jail: nginx content server (WordPress)
```

## 発生した問題

### 症状
1. **HTTP/1.1**: 正常に動作
2. **HTTP/2**: パケットが流れない
3. **旧SSL対応ブラウザ（w3m等）**: アクセス可能
4. **モダンブラウザ**: データが送信されない

### 詳細な現象
- nginx のログ上ではエラーが記録されていない
- パケットは正常に送信されているように見える
- リバースプロキシを迂回した直接接続では一時的にHTTP/2が動作
- 時間経過とともに直接接続でも接続不能になる
- nginx バージョン（1.26〜1.29）を変更しても同様の症状

## トラブルシューティング手順

### 1. nginx設定の確認
```nginx
# リバースプロキシ設定例
server {
    listen 443 ssl http2;
    server_name example.com;
    
    # SSL/TLS設定
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    
    # HTTP/2設定
    http2 on;  # または listen 443 ssl http2;
    
    location / {
        proxy_pass http://backend_jail_ip:80;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 2. PF設定の確認
```pf
# 基本的なリダイレクト設定
rdr pass on $ext_if proto tcp from any to $public_ip port 443 -> $jail_ip port 443
```

### 3. 検証手順
1. **HTTP/1.1での接続テスト**
   ```bash
   curl -v -H "Host: example.com" https://your-server.com/
   ```

2. **HTTP/2での接続テスト**
   ```bash
   curl -v --http2 -H "Host: example.com" https://your-server.com/
   ```

3. **w3mでの接続確認**
   ```bash
   w3m -dump https://your-server.com/
   ```

## 考察と仮説

### 主要な仮説：PFによるパケット処理の問題

1. **UDPフォワーディング処理**: 
   - PFがUDPパケットのソースIPアドレス書き換えを実行
   - パケット整列処理が継続的に行われている

2. **QUICプロトコルとの干渉**:
   - QUICはセッション管理を行わないプロトコル
   - PFの「内部的なセッション管理」がQUICプロトコルに干渉する可能性

3. **HTTP/2とTCPの関係**:
   - HTTP/2はTCPベースだが、より複雑なストリーム管理を行う
   - PFのパケット加工処理がHTTP/2のストリーム整合性に影響

### FreeBSD PFの特性
- CG-NATとは異なる動作特性
- 近年のLinux実装と比較してVRF等の機能に制限
- コンシューマレベルでのルータ利用が困難になりつつある

## 対処方針

### 短期的対策
1. **HTTP/1.1の継続使用**
2. **リバースプロキシの設定最適化**
3. **PF設定の見直し**

### 中長期的検討事項
1. **ファイアウォール解決策の検討**
   - pfSenseやOPNsenseの導入検討
   - Linux iptables/netfilterへの移行検討

2. **アーキテクチャの見直し**
   - ロードバランサの導入
   - CDNの活用

## スクリーンショット挿入箇所

以下の箇所でスクリーンショットを追加することをお勧めします：

1. **nginx設定ファイルの内容**
   - `/usr/local/etc/nginx/nginx.conf`
   - サイト固有の設定ファイル

2. **PF設定の確認**
   - `pfctl -sr` の出力
   - `pfctl -sn` の出力

3. **接続テスト結果**
   - `curl -v --http2` の出力
   - ブラウザ開発者ツールのNetwork タブ

4. **ログファイル**
   - nginx access.log
   - nginx error.log
   - システムログ

## まとめ

FreeBSD jail環境でのHTTP/2接続問題は、主にPFによるパケット処理とHTTP/2プロトコルの相互作用が原因と考えられます。immediate solutionとしてはHTTP/1.1での運用を継続し、中長期的にはインフラ構成の見直しを検討することが推奨されます。

---

**作業日**: 2025年9月8日  
**環境**: FreeBSD + jail + nginx + PF  
**問題**: HTTP/2接続不能  
**ステータス**: 調査継続中