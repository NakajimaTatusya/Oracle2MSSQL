# 概要

Oracleコレクション型を、SQL Serverのカーソル、ローカルテンポラリテーブル、TABLE型変数へ変換する。  
また、Oracleコレクション型を使用した、LIMIT FORALLについても上記へ置き換える。

# 用途

* 問い合わせ結果を複数行読み込み、加工しながら追加、更新を行う。

# 使い分け

* 問い合わせ結果を1レコードずつ加工、他のテーブルを参照などしたい場合、「カーソル」を使用する。  
* Oracleの BULK COLLECT LIMIT ～ FORALL をSQL Serverで代替する場合、「ローカルテンポラリテーブル」を使用する。
* 参照データが少量の場合、「TABLE変数」を使用する。

# SQL

## 変換前

```SQL
～
IS
    TYPE recTypeSrc IS RECORD
    (
        ID TBL_SOURCE.ID%TYPE,
        SITUATION TBL_SOURCE.SITUATION%TYPE
    );
    TYPE tblTypeSrc IS TABLE OF recTypeSrc INDEX BY BINARY_INTEGER;
    tblSrc tblTypeSrc;
    TYPE curType IS REF CURSOR;
    curSrc curType;
    i NUMBER;
    query VARCHAR2(3000) := 'SELECT ID, SITUATION FROM TBL_SOURCE';
BEGIN
    OPEN curSrc FOR query;

    FETCH curSrc BULK COLLECT INTO tblSrc;

    CLOSE curSrc;

    i := tblSrc.FIRST;
    WHILE i IS NOT NULL LOOP
        --加工処理
        i := tblSrc.NEXT(i);
    END LOOP;
END;
```

## 変換後

### カーソル

```SQL
DROP PROCEDURE STATIC_CURSOR_DEMO
GO

CREATE OR ALTER PROCEDURE STATIC_CURSOR_DEMO
AS
BEGIN
    DECLARE @ret_id INT, @ret_situ NVARCHAR(10);
    DECLARE @opedate DATETIME2;

    SET @opedate = SYSDATETIME();

    DELETE FROM RESULT_TABLE;

    DECLARE curStatic CURSOR LOCAL FORWARD_ONLY STATIC READ_ONLY
        FOR SELECT ID, SITUATION FROM TBL_SOURCE;

    OPEN curStatic

    WHILE 1 = 1
    BEGIN
        FETCH NEXT FROM curStatic INTO @ret_id, @ret_situ;
        IF  @@FETCH_STATUS <> 0
        BEGIN
            BREAK;
        END
        
        SET @ret_situ = CONCAT(@ret_situ, '笛吹市');/*値加工*/

        INSERT RESULT_TABLE SELECT @ret_id, @ret_situ, @opedate, @opedate;
    END;

    CLOSE curStatic;
    DEALLOCATE curStatic;

    RETURN(0);
END
GO
```

```SQL
DECLARE @return_code INT;
EXEC @return_code = STATIC_CURSOR_DEMO;
PRINT @return_code;
```

### ローカルテンポラリテーブル

```SQL
DROP PROCEDURE LOCAL_TEMPTABLE_DEMO
GO

CREATE OR ALTER PROCEDURE LOCAL_TEMPTABLE_DEMO
AS
BEGIN
    /* ローカルテンポラリテーブルが存在しない場合は作る */
    IF OBJECT_ID(N'tempdb..#LoTempSource', N'U') IS NULL
    BEGIN
        CREATE TABLE #LoTempSource
        (
            ID          INT NOT NULL PRIMARY KEY,
            SITUATION   NVARCHAR(10),
            DATEADDED   DATETIME2 DEFAULT SYSDATETIME(),
            UPDATED     DATETIME2
        );
    END
    ELSE
    BEGIN
        /* すでにローカルテンポラリテーブルがあったら中身を消す */
        DELETE FROM #LoTempSource;
    END

    DECLARE @updateSitu NVARCHAR(10);
    DECLARE @opedate DATETIME2;

    SET @opedate = SYSDATETIME();

    DELETE FROM RESULT_TABLE;

    /* ローカルテンポラリテーブルに更新対象の全レコードを読み込む */
    /* Oracleのカーソルオープン～BULK COLLECT LIMIT に相当 */
    INSERT #LoTempSource SELECT ID, SITUATION, @opedate, null FROM TBL_SOURCE;

    DECLARE curUpdate CURSOR LOCAL
        FOR SELECT SITUATION FROM #LoTempSource ORDER BY ID
        FOR UPDATE OF SITUATION;

    OPEN curUpdate

    /* このループがOracleコレクションを使用して更新しているところに相当 */
    WHILE 1 = 1
    BEGIN
        FETCH NEXT FROM curUpdate INTO @updateSitu;
        IF  @@FETCH_STATUS <> 0
        BEGIN
            BREAK;
        END

        SET @updateSitu = CONCAT(@updateSitu, '甲府市');/*値加工*/

        /* ローカルテンポラリテーブルの列を更新カーソルで更新する */
        UPDATE #LoTempSource SET SITUATION = @updateSitu WHERE CURRENT OF curUpdate;
    END;

    CLOSE curUpdate;
    DEALLOCATE curUpdate;

    /* 加工した全レコードを一括更新 FOALLに相当 */
    INSERT RESULT_TABLE SELECT * FROM #LoTempSource;

    /* 使い終わったローカルテンポラリテーブルを削除する。 */
    IF OBJECT_ID(N'tempdb..#LoTempSource', N'U') IS NOT NULL
        DROP TABLE #LoTempSource;

    RETURN(0);
END
GO
```

```SQL
DECLARE @return_code INT;
EXEC @return_code = LOCAL_TEMPTABLE_DEMO;
PRINT @return_code;
```

### TABLE変数

```SQL
DROP PROCEDURE TABLE_TYPE_DEMO
GO

CREATE OR ALTER PROCEDURE TABLE_TYPE_DEMO
(
    @param1 INT,
    @param2 INT
)
AS
BEGIN
    DECLARE @tblSrc TABLE
    (
        ID          INT NOT NULL PRIMARY KEY,
        SITUATION   NVARCHAR(10),
        DATEADDED   DATETIME2 DEFAULT SYSDATETIME(),
        UPDATED     DATETIME2
    )

    DECLARE @opedate DATETIME2;

    SET @opedate = SYSDATETIME();

    DELETE FROM RESULT_TABLE;

    INSERT @tblSrc 
        SELECT ID, SITUATION, @opedate AS DATEADDED, @opedate AS UPDATED FROM TBL_SOURCE WHERE ID >= @param1 AND ID <= @param2

    INSERT RESULT_TABLE SELECT * FROM @tblSrc;

    RETURN(0);
END
GO
```

```SQL
DECLARE @return_code INT;
EXEC @return_code = TABLE_TYPE_DEMO 1, 1000;
PRINT @return_code;
```

#　補足事項

* 行セットをカーソルで処理する場合は、「FORWARD_ONLY STATIC READ_ONLY」を指定して最適化する。
* TABLE変数は、少量のデータならばローカルテンポラリテーブルと同程度の処理速度が期待できる。
* グローバルテンポラリテーブルは、スコープが広くライフサイクルも長いため使用しない。
* BULK COLLECT LIMITをSQL Serverで実現する方法はないため対応しない。

# 関連エラー

* エラーコード：O2SS0157への対応
* エラーコード：O2SS0174への対応
* エラーコード：O2SS0345への対応
* エラーコード：O2SS0408への対応
* エラーコード：O2SS0472への対応
