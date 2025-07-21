# 動的AZとCIDR計算ガイド

[English](../en/05_calculate_az_and_cidr_dynamically.md) | [繁體中文](../zh-tw/05_calculate_az_and_cidr_dynamically.md) | [日本語](../ja/05_calculate_az_and_cidr_dynamically.md) | [インデックスに戻る](../README.md)

この文書では、Nebulettaプロジェクトにおける動的Availability Zone（AZ）選択とCIDRブロック自動計算の実装方法について説明します。

---

## 背景
- 実験日：2025/07/21
- 難易度：🤬
- 説明：初期の実装では手動計算と設定に大きく依存しており、理想的な解決策ではありませんでした。

---

### 従来のハードコード方式
```hcl
# 古い実装方法
azs = ["ap-northeast-1a", "ap-northeast-1c"]
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
```

**問題点：**
- リージョン間で使用できない（各リージョンでAZが異なる）
- AZリストの手動メンテナンスが必要
- CIDRブロックの手動計算が必要
- 設定ミスが発生しやすい

---

## 解決策：動的計算

### 1. 自動AZ検出

```hcl
# data.tf
data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

# locals.tf
available_azs = slice(
  data.aws_availability_zones.available.names, 
  0, 
  min(var.max_azs, length(data.aws_availability_zones.available.names))
)
```

**重要概念：**
- `state = "available"` - 利用可能なAZのみを選択
- `opt-in-status = "opt-in-not-required"` - 追加申請不要のAZのみを選択
- `min()`関数で実際に利用可能なAZ数を超えないことを保証

### 2. 動的CIDR計算

#### ステップ1：VPC CIDR分解
```hcl
vpc_cidr_prefix = split("/", var.vpc_cidr)[0]  # "10.0.0.0"
vpc_cidr_mask   = tonumber(split("/", var.vpc_cidr)[1])  # 16
```

#### ステップ2：IPアドレスセグメント処理
```hcl
# "10.0.0.0" を ["10", "0", "0", "0"] に分割
split(".", local.vpc_cidr_prefix)

# 最初の2セグメント ["10", "0"] を取得
slice(split(".", local.vpc_cidr_prefix), 0, 2)

# "10.0" に再結合
join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))
```

#### ステップ3：サブネットCIDR生成
```hcl
# パブリックサブネット: 10.x.1.0/24, 10.x.2.0/24, ...
public_subnet_cidrs = [
  for i in range(length(local.available_azs)) :
  "${join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))}.${i + 1}.0/24"
]

# プライベートサブネット: 10.x.11.0/24, 10.x.12.0/24, ...
private_subnet_cidrs = [
  for i in range(length(local.available_azs)) :
  "${join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))}.${i + 11}.0/24"
]
```

---

## 関数詳細説明

### Terraform関数リファレンス

| 関数 | 用途 | 例 |
|------|------|-----|
| `slice(list, start, end)` | リストから部分を抽出 | `slice(["a","b","c"], 0, 2)` → `["a","b"]` |
| `split(separator, string)` | 文字列分割 | `split("/", "10.0.0.0/16")` → `["10.0.0.0", "16"]` |
| `join(separator, list)` | リスト結合 | `join(".", ["10","0"])` → `"10.0"` |
| `tonumber(string)` | 文字列を数値に変換 | `tonumber("16")` → `16` |
| `min(a, b)` | より小さい値を取得 | `min(4, 3)` → `3` |
| `length(list)` | リストの長さを取得 | `length(["a","b"])` → `2` |
| `range(n)` | 数値シーケンス生成 | `range(3)` → `[0, 1, 2]` |

### 完全計算例

前提条件：
- `var.vpc_cidr = "10.0.0.0/16"`
- `var.max_azs = 3`
- ap-northeast-1で利用可能なAZ：`["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]`

**計算プロセス：**

1. **AZ選択：**
   ```hcl
   available_azs = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
   ```

2. **CIDR分解：**
   ```hcl
   vpc_cidr_prefix = "10.0.0.0"
   join(".", slice(split(".", "10.0.0.0"), 0, 2)) = "10.0"
   ```

3. **パブリックサブネット計算：**
   ```hcl
   i=0: "10.0.1.0/24"   # ap-northeast-1a
   i=1: "10.0.2.0/24"   # ap-northeast-1c  
   i=2: "10.0.3.0/24"   # ap-northeast-1d
   ```

4. **プライベートサブネット計算：**
   ```hcl
   i=0: "10.0.11.0/24"  # ap-northeast-1a
   i=1: "10.0.12.0/24"  # ap-northeast-1c
   i=2: "10.0.13.0/24"  # ap-northeast-1d
   ```

---

## メリット総括

### ✅ 解決された問題
1. **クロスリージョン互換性** - 任意のAWSリージョンのAZ構成に自動適応
2. **設定複雑度の軽減** - `max_azs`パラメータの指定のみで済む
3. **人的ミスの削減** - 自動CIDR計算により競合を回避
4. **柔軟な制御** - 要件に応じてAZ数を調整可能
5. **フォルトトレラント機能** - AZ数不足の場合を自動処理

### ✅ 実際の効果
- **メンテナンス性向上** - ハードコード設定を削減
- **デプロイメント柔軟性** - 同一コードベースで異なるリージョンに対応
- **スケーラビリティ** - AZ数の調整が簡単
- **一貫性** - CIDR割り当てルールの統一

---

## 特殊ケース説明

### ap-northeast-1の実際の状況
- 理論上は4つのAZ（a、b、c、d）
- 実際には3つのみ利用可能（ap-northeast-1bは廃止済み）
- システムが自動検出して使用：`["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]`

### CIDR割り当てルール
- **VPC CIDR**: `10.0.0.0/16`（65536個のIP）
- **パブリックサブネット**: `10.0.1.0/24`、`10.0.2.0/24`、...（各256個のIP）
- **プライベートサブネット**: `10.0.11.0/24`、`10.0.12.0/24`、...（各256個のIP）
- **予約スペース**: `10.0.0.0/24`、`10.0.4-10.0/24`、`10.0.14+.0/24`を他の用途で利用可能

この設計により、ネットワークアーキテクチャの柔軟性と保守性を確保しています。
