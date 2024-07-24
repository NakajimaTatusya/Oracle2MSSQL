# 概要

LPAD関数はSQL Serverに存在しない処理なので、する場合はRIGHT関数を使用する。

# 用途

* ゼロパディング

# SQL

## 変換前

```SQL
SELECT LPAD('12', 3, '') FROM DUAL
```

## 変換後

```SQL
SELECT RIGHT(CONCAT('000', '12'), 3)
```

#　補足事項

表示用途であればパディングは画面で行うべき。

# 関連事項

* 文字列の結合はCONCATを使用する
