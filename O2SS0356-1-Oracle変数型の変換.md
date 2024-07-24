# 概要

* Oracleの変数宣言で使用している型は、SQL Serverの型に手動で変換する。
* Oracleの変数宣言で%ROWTYPEを使用している個所は、自動変換される。

# 用途

* 適切な型宣言を行う

# SQL

## 整数型

* 結果セット件数などの整数を格納する変数に浮動小数点型を使用している場合

### 変換前

```SQL
recordCount NUMBER;
```

### 変換後

```SQL
recordCount INT;
```

## 2値のみの整数

* 0,1のみを格納する変数宣言

### 変換前

```SQL
result BINARY_INTEGER;
```

### 変換後

```SQL
result BIT;
```

## 容量を指定しない可変長文字列

* 容量を関係するテーブル列に合わせて指定する。

### 変換前

```SQL
charstring VARCHAR2;
```

### 変換後

```SQL
charstring VARCHAR(100);
```

## 固定長を格納する変数に可変長型

* 固定長を扱う変数に可変長型を使用している場合は固定長型を使用する。

### 変換前

```SQL
eigyocd VARCHAR2;
```

### 変換後

```SQL
eigyocd CHAR(1);
```
