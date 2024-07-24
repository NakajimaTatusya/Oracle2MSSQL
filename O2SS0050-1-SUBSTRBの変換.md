# 概要

OracleのSUBSTRB関数は、SQL Serverに存在しない。SQL Serverの可変長文字列はNVARCHARを使用するのでSubstring関数を使用する。

# 用途

* 文字列の切り出し

# SQL

## 変換前

```SQL
SELECT SUBSTRB('あaいiうuえeおo', 3, 1) FROM DUAL;
--結果 a
```

## 変換後

```SQL
DECLARE @targetstring NVARCHAR(max);
SET @targetstring = 'あaいiうuえeおo';
SELECT Substring(@targetstring, 2, 1);
--結果 a
```
