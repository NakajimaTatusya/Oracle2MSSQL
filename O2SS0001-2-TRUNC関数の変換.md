# 概要

日付のTRUNC関数はSQL Serverに存在しない処理なので、する場合はEOMONTH関数を使用する。

# 用途

* 日時の切り捨て、または月初日の取得

# SQL

## 変換前

```SQL
SELECT TRUNC(SYSDATE, 'MONTH') FROM DUAL
```

## 変換後

```SQL
SELECT DATEADD(month, DATEDIFF(month, 0, SYSDATETIME()), 0);
```

#　補足事項

月初日は、DATEDIFFで1900/1/1からシステム日付までで何か月経過したか取得し、DATEADDで1900/1/1にその経過月数を加算することで求める。
