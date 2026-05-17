# VPING / EncryptSecureDEC PNG 暗号化ファイル仕様書

## 1. 概要

本フォーマットは、任意のバイナリファイルを暗号化し、PNG画像形式のコンテナとして保存する独自ファイル形式である。

主な特徴は以下の通り。

- AES-GCMによる暗号化
- PBKDF2によるパスワードベース鍵導出
- LZMA圧縮
- Reed-Solomon ECCによる軽度破損耐性
- SHA-256による整合性検証
- PNGメタデータによる形式情報保持
- ブロックチェーン風の操作履歴保持

保存拡張子は `.png` である。

---

## 2. ファイル全体構造

```text
PNG Container
├─ PNG RGB Pixel Data
│   └─ ECC付きLZMA圧縮データ
│
└─ PNG Text Metadata
    └─ EncryptSecureDEC
```

PNG画像のピクセルデータ部分に暗号化済みデータを格納し、PNGメタデータに復元・検証に必要な情報を保存する。

---

## 3. PNGメタデータ仕様

PNGのテキストメタデータとして、以下のキーを使用する。

```text
EncryptSecureDEC
```

このキーの値にはJSON文字列を格納する。

### 3.1 メタデータJSON構造

```json
{
  "compressed_length": 12345,
  "protected_length": 12377,
  "text_metadata": "txt",
  "encoder": "EncryptSecureDEC PNG v3",
  "width": 256,
  "height": 256,
  "ecc_bytes": 32,
  "original_sha256": "...",
  "compressed_sha256": "..."
}
```

### 3.2 メタデータ項目

| 項目 | 内容 |
|---|---|
| compressed_length | LZMA圧縮後のデータサイズ |
| protected_length | ECC付与後のデータサイズ |
| text_metadata | 元ファイルの拡張子 |
| encoder | エンコーダ名・フォーマットバージョン |
| width | 保存PNGの横幅 |
| height | 保存PNGの高さ |
| ecc_bytes | Reed-Solomon ECCのバイト数 |
| original_sha256 | 復元後の元データSHA-256 |
| compressed_sha256 | LZMA圧縮後データのSHA-256 |

---

## 4. PNG内部データ構造

PNGのRGBピクセルには、以下の順序で処理されたデータが格納される。

```text
暗号化データ + ブロックチェーン履歴
↓
LZMA圧縮
↓
Reed-Solomon ECC付与
↓
RGBピクセル化
↓
PNG保存
```

---

## 5. 復元後データ構造

LZMA展開後のデータは以下の構造を持つ。

```text
[ encrypted_data ]
+
[ BLOCKCHAIN_HEADER ]
+
[ blockchain_json ]
```

### 5.1 BLOCKCHAIN_HEADER

ブロックチェーン履歴の開始位置を示す固定ヘッダ。

```python
b'BLOCKCHAIN_DATA_START\n'
```

---

## 6. encrypted_data 構造

暗号化データ部分は以下のバイト構造である。

```text
offset  size    内容
0x00    16      salt
0x10    12      AES-GCM nonce
0x1C    可変     ciphertext
末尾16   16      AES-GCM tag
```

### 6.1 暗号仕様

| 項目 | 値 |
|---|---|
| 暗号方式 | AES-256-GCM |
| 鍵導出 | PBKDF2 |
| PBKDF2反復回数 | 100,000 |
| salt長 | 16 bytes |
| nonce長 | 12 bytes |
| tag長 | 16 bytes |
| 鍵長 | 32 bytes |

---

## 7. ブロックチェーン履歴仕様

ブロックチェーン履歴はJSON配列として保存される。

### 7.1 Block構造

```json
{
  "timestamp": "2026-01-01 00:00:00.000000+00:00",
  "data": "...",
  "previous_hash": "...",
  "operation_type": "Encrypt",
  "file_hash": "...",
  "user": "...",
  "memo": "...",
  "hash": "..."
}
```

### 7.2 各フィールド

| 項目 | 内容 |
|---|---|
| timestamp | UTCタイムスタンプ |
| data | ブロックに格納するデータ |
| previous_hash | 前ブロックのハッシュ |
| operation_type | 操作種別。Encrypt または Decrypt |
| file_hash | ciphertextのSHA-256 |
| user | 操作者名 |
| memo | 操作メモ |
| hash | 当該ブロックのSHA-256 |

### 7.3 ブロックハッシュ計算

以下の項目を文字列化し、UTF-8で連結してSHA-256を計算する。

```text
timestamp
data
previous_hash
operation_type
file_hash
user
memo
```

---

## 8. ECC仕様

| 項目 | 値 |
|---|---|
| 方式 | Reed-Solomon |
| ECCバイト数 | 32 bytes |
| 対象 | LZMA圧縮済みデータ |

ECCは圧縮後のデータに付与される。
そのため、軽度のPNGピクセル破損や一部バイト破損に対して復元可能性を持つ。

---

## 9. 圧縮仕様

| 項目 | 値 |
|---|---|
| 圧縮方式 | LZMA |
| 圧縮対象 | encrypted_data + blockchain_data |

---

## 10. RGB格納仕様

ECC付与後のバイナリを3バイト単位でRGBピクセルへ変換する。

```text
byte[0] → R
byte[1] → G
byte[2] → B
```

データ長が3の倍数でない場合、不足分は `0x00` で埋める。

画像サイズは必要ピクセル数をもとに、概ね正方形になるよう算出される。

```text
width  = ceil(sqrt(pixel_count))
height = ceil(pixel_count / width)
```

---

## 11. 復号フロー

```text
PNG読込
↓
PNGメタデータ取得
↓
RGBピクセルからECC付きデータを取得
↓
ECC復元
↓
LZMA展開
↓
BLOCKCHAIN_HEADERで暗号データと履歴を分離
↓
salt / nonce / ciphertext / tag を分離
↓
PBKDF2で鍵導出
↓
AES-GCM復号およびtag検証
↓
元ファイル復元
↓
復号履歴をブロックチェーンへ追加
↓
PNGを更新保存
```

---

## 12. 整合性検証仕様

検証では以下を確認する。

- 復元後データのSHA-256が `original_sha256` と一致するか
- 圧縮データのSHA-256が `compressed_sha256` と一致するか
- AES-GCM tag検証に成功するか
- ブロックチェーンの `previous_hash` が連鎖しているか
- 各ブロックの再計算ハッシュが保存ハッシュと一致するか

---

## 13. フォーマットバージョン

| encoder | 内容 |
|---|---|
| EncryptSecureDEC PNG | v1。ECCなし |
| EncryptSecureDEC PNG v2 | v2。内部データ側ECC想定 |
| EncryptSecureDEC PNG v3 | v3。LZMA圧縮後にECC付与 |

現行仕様では `EncryptSecureDEC PNG v3` が使用される。

---

## 14. 拡張子保持仕様

元ファイルの拡張子はPNGメタデータの `text_metadata` に保存される。

例:

```json
{
  "text_metadata": "pdf"
}
```

復号時、この値を使用して復元ファイル名を生成する。

---

## 15. 復号後ファイル名仕様

設定 `output_suffix` により復号後ファイル名が変化する。

### output_suffix = true の場合

```text
sample.png
↓
sample.txt
```

### output_suffix = false の場合

```text
sample.png
↓
sample.txt_decrypted
```

---

## 16. セキュリティ特性

### 16.1 機密性

AES-256-GCMによりファイル本体を暗号化する。
パスワードからPBKDF2により鍵を導出するため、同じパスワードでもsaltにより異なる鍵が生成される。

### 16.2 改竄検知

以下の仕組みにより改竄を検出できる。

- AES-GCM tag
- original_sha256
- compressed_sha256
- ブロックチェーン風履歴ハッシュ

### 16.3 耐破損性

Reed-Solomon ECCにより、軽度のバイト破損を復元できる可能性がある。
ただし、ECCで復元できない規模の破損がある場合、復号は失敗する。

---

## 17. 実質的なファイルレイアウト

```text
PNG
└─ RGB Pixel Data
   └─ Reed-Solomon Protected Data
      └─ LZMA Compressed Data
         ├─ AES-GCM encrypted payload
         │  ├─ salt
         │  ├─ nonce
         │  ├─ ciphertext
         │  └─ tag
         │
         ├─ BLOCKCHAIN_DATA_START\n
         └─ blockchain JSON
```

---

## 18. 注意事項

- PNGとして表示可能だが、通常の画像として意味のある内容を持つとは限らない。
- PNG最適化ツールや画像編集ソフトで再保存すると、ピクセルデータやメタデータが変更され、復号不能になる可能性がある。
- メタデータ `EncryptSecureDEC` が失われると復元に必要な長さ情報やハッシュ情報を取得できない。
- パスワードを忘れた場合、AES-GCM暗号化データの復号は困難である。
- ブロックチェーン履歴は改竄検知用であり、分散型ブロックチェーンではない。

---

## 19. まとめ

本形式は、PNG画像をコンテナとして利用し、暗号化ファイル本体・整合性検証情報・操作履歴をまとめて保持する独自暗号化ファイル形式である。

実質的には、以下の性質を持つ。

```text
画像形式の外観を持つ、暗号化コンテナ兼監査ログ付きファイル形式
```
