# 概要

OracleのTRIM関数を、MSSSQL2017以降ではTRIM関数を使用できるのでこちらに変換する。

# 用途

* 空文字をコード未設定としているところがあり、その判定用にTRIMを使用する。

# SQL

## 変換前

```SQL
WHEN TRIM(列名) IS NULL THEN
```

## 変換後

```SQL
--VARCAHR、CHAR型列で、NULL許容であった場合
WHEN TRIM(列名) IS NULL OR TRIM(列名) = '' THEN
```

```SQL
--VARCHAR、CHAR型列で、NOT NULLであった場合
WHEN TRIM(列名) = '' THEN
```

#　補足事項

TRIM関数にNULLを与えると、NULLを返す。  
TRIM関数に' '（空白）を与えると、長さゼロの文字列を返却する。  
Oracleでは長さゼロの文字列はNULLとされるが、SQL ServerではNULLと区別される。
