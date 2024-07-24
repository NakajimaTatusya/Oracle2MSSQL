# 概要

OracleのSUBSTR関数は、SQL ServerのSUBSTRING関数に変換する。

# 用途

* 文字列の切り出し

# SQL

## 変換前

```SQL
variable := SUBSTR(targetString, 0, INSTR(targetString, ';'));
```

## 変換後

```SQL
DECLARE @targetString NVARCHAR(20) = N'Hellow World;';
SELECT SUBSTRING(@targetString, 0, CHARINDEX(';', @targetString));
```

# 関連事項

* Oracleの「INSTR関数」とともに使用されている場合は、SQL Serverの「CHARINDEX関数」を使用する。

# 関連エラー

* エラーコード：O2SS0557への対応
* 警告コード：O2SS0273への対応
