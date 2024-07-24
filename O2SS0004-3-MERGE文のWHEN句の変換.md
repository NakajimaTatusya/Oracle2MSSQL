# 概要

MERGEステートメントのINSERT句およびUPDATE句の代入先列名にテーブル別名.列名が使用されているためコンパイルエラーとなっている。

# 用途

* MERGEステートメント

# SQL

## 変換前

```SQL
MERGE INTO TBL_EXAMPLE A
    USING
        (SELECT 1 AS ID, 'test' AS CAPTION) AS B
        ON
        (A.ID = B.ID)
    WHEN NOT MATCHED THEN
        INSERT (
            A.ID,/*エラーとなる記述*/
            A.CAPTION/*エラーとなる記述*/
        )
        VALUES (
            B.ID,
            B.CAPTION
        )
    WHEN MATCHED THEN
        UPDATE SET
            A.CAPTION = B.ID/*エラーとなる記述*/
;
```

## 変換後

```SQL
MERGE INTO TBL_EXAMPLE A
    USING
        (SELECT 1 AS ID, 'test' AS CAPTION) B
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
