# Network ACLがPrivate Subnet通信に与える影響のデバッグ経験

[English](../en/01_network_acl_private_subnet_troubleshooting.md) | [繁體中文](../zh-tw/01_network_acl_private_subnet_troubleshooting.md) | [日本語](01_network_acl_private_subnet_troubleshooting.md) | [索引に戻る](../README.md)

---

## 背景
- 実験日時: 2025/05/21
- 難易度: 🤬🤬🤬
- 説明: AWS VPC環境において、関連設定がすべて正しく見えるにも関わらず、private subnet内のEC2インスタンスが外部ネットワークと正常に通信できない状況。

## 観察された症状

- EC2はSession Managerを通じて正常にアクセス可能
- `ping www.google.com`が応答なし
- `curl https://google.com`がハングして接続完了できない

## デバッグプロセス

### 第1段階：インフラ設定の確認

以下の項目がすべて正しく設定されていることを確認：

| 項目 | 状態 | 説明 |
|------|------|------|
| EC2 Instance | ✅ | Public IPなし、Private Subnetに正しく配置 |
| Private Subnet | ✅ | 対応するRoute Tableあり |
| Route Table | ✅ | `0.0.0.0/0` ➝ NAT Gateway設定済み |
| NAT Gateway | ✅ | Public SubnetにElastic IPで展開済み |
| Security Group | ✅ | Outboundが`0.0.0.0/0` all traffic |

### 第2段階：DNS解決テスト

`nslookup google.com`を実行して正常な結果を取得：

```bash
Server:         10.0.0.2
Address:        10.0.0.2#53

Non-authoritative answer:
Name:   google.com
Address: 142.251.42.142
Name:   google.com
Address: 2404:6800:4004:827::200e
```

**結論**：DNS解決は正常で、VPCのDNS設定と`/etc/resolv.conf`に問題なし。

### 第3段階：可能性のある原因の排除

DNSは正常だがHTTPS接続が失敗するため、可能性のある原因：

1. ❌ NAT GatewayがInternet Gatewayに接続されていない
2. ❌ NAT GatewayのあるPublic SubnetのRoute Tableに`0.0.0.0/0` ➝ IGWがない
3. ✅ **NACL（Network ACL）が接続をブロック** ← 真の問題！
4. ❌ EC2のSource/Dest Checkが無効化されていない（NAT Instanceのみ適用）

## 根本原因

### Network ACL設定エラー

private subnetにバインドされたNetwork ACLルールを確認：

**Inboundルール（問題あり）：**
| Rule # | Source | Protocol | Port | Allow/Deny |
|--------|--------|----------|------|------------|
| 100 | 10.0.0.0/16 | All | All | ✅ Allow |
| * | 0.0.0.0/0 | All | All | ❌ **Deny** ← ここが問題！|

**Outboundルール（正常に見えるが落とし穴あり）：**
| Rule # | Destination | Protocol | Port | Allow/Deny |
|--------|-------------|----------|------|------------|
| 100 | 0.0.0.0/0 | All | All | ✅ Allow |
| * | 0.0.0.0/0 | All | All | ❌ **Deny** ← まだ有効！|

## 問題の原理解析

### ICMP（Ping）のワークフロー

1. **Outbound ICMP Request**：
   - EC2が外部アドレスにpingパケットを送信
   - Outbound Rule 100: `0.0.0.0/0 Allow`に適合
   - パケットは正常に送信される

2. **Inbound ICMP Reply**：
   - 外部サーバーがping応答を返す
   - 送信元IPがVPC CIDR（`10.0.0.0/16`）ではない
   - Inbound Rule `* 0.0.0.0/0 Deny`によってブロック
   - **応答パケットが入れず、ping失敗**

### 他のプロトコルへの影響

同じ問題が以下にも影響：
- HTTPS接続（port 443）
- HTTP接続（port 80）  
- 外部応答が必要なすべての通信

## 解決策

### 追加が必要なInboundルール

private subnet内のEC2が外部ネットワークと正常に通信できるようにするため、Network ACLに以下のinboundルールを追加する必要がある：

1. **ICMP応答を許可**：
   ```hcl
   ingress {
     protocol   = "1"  # ICMP
     rule_no    = 80
     action     = "allow"
     cidr_block = "0.0.0.0/0"
     from_port  = 0
     to_port    = 0
     icmp_type  = -1
     icmp_code  = -1
   }
   ```

2. **Ephemeral Port応答トラフィックを許可**：
   ```hcl
   ingress {
     protocol   = "6"  # TCP
     rule_no    = 90
     action     = "allow"
     cidr_block = "0.0.0.0/0"
     from_port  = 1024
     to_port    = 65535
   }
   ```

3. **HTTPS応答を許可**：
   ```hcl
   ingress {
     protocol   = "6"  # TCP
     rule_no    = 100
     action     = "allow" 
     cidr_block = "0.0.0.0/0"
     from_port  = 443
     to_port    = 443
   }
   ```

## 重要な学習ポイント

### Network ACL vs Security Group

| 特徴 | Network ACL | Security Group |
|------|-------------|----------------|
| 動作レベル | Subnetレベル | Instanceレベル |
| 状態 | **ステートレス** | ステートフル |
| 応答処理 | 応答トラフィックを明示的に許可する必要 | 応答トラフィックを自動許可 |
| デフォルト動作 | デフォルトですべて拒否 | デフォルトでinbound拒否、outbound許可 |

### ステートレスの重要性

- **ステートレス**はNetwork ACLが接続状態を記憶しないことを意味
- Outbound許可はInbound応答も許可されることを意味しない
- **双方向ルールを明示的に設定する必要**

### Ephemeral Portの重要性

- TCP接続確立時、クライアントはランダムな高ポート番号（1024-65535）を使用
- サーバー応答はこれらの高ポートに送信される
- inboundでこれらのポートを許可しないと、接続が失敗

## 予防措置

1. **Network ACL設計時は双方向通信を優先的に考慮**
2. **ephemeral port範囲を無視しない**
3. **テスト時はoutboundとinboundトラフィック両方を検証**
4. **Network ACLのステートレス特性を記憶**

## 結論

今回のデバッグ経験により、AWSネットワーク設定において、Network ACLは見落とされがちだが影響が大きいコンポーネントであることが再確認された。特にprivate subnetの設定では、すべての可能な応答トラフィックを慎重に考慮して、サービスが正常に動作することを確保する必要がある。

**記憶せよ**：pingを送信できることは応答を受信できることを意味しない - ステートレスなNetwork ACLには明示的な双方向ルール設定が必要！ 