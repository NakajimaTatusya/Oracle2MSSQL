# 概要

SQL ServerにROWIDは存在しないので、ROWIDでのテーブル自己結合は該当するテーブルの主キーを結合条件とする。

# 用途

* テーブルの自己結合

# SQL

## 変換前

```SQL
MERGE INTO TBL_EXAMPLE A
    USING
        (SELECT ROWID, ID, 'test' AS CAPTION FROM TBL_EXAMPLE WHERE ID = 1) AS B
        ON
        (A.ROWID = B.ROWID)
    WHEN NOT MATCHED THEN
        INSERT (
            ID,
            CAPTION
        )
        VALUES (
            B.ID,
            B.CAPTION
        )
    WHEN MATCHED THEN
        UPDATE SET
            CAPTION = B.ID
;
```

## 変換後

```SQL
MERGE INTO TBL_EXAMPLE A
    USING
        (SELECT ID, 'test' AS CAPTION FROM TBL_EXAMPLE WHERE ID = 1) B
        ON
        (A.ID = B.ID)
    WHEN NOT MATCHED THEN
        INSERT (
            ID,
            CAPTION
        )
        VALUES
        (
            B.ID,
            B.CAPTION
        )
    WHEN MATCHED THEN
        UPDATE SET
            CAPTION = B.CAPTION
;
```

# 関連エラー

* エラーコード：O2SS0401への対応
