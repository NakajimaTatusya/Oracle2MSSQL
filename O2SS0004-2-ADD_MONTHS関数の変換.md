# 概要

ADD_MONTHS関数は、SQL Serverに存在しない関数のためDATEADD関数へ変換する。

# 用途

* 主に検索条件に使用する日付の加算を行う。

# SQL

## 変換前

```SQL
SELECT ADD_MONTHS(日付変数, 12)
```

## 変換後

```SQL
SELECT DATEADD(m, 12, 日付変数)
```
