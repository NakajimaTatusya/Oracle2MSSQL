# 概要

Oracleでは「PIPE ROW」により表を作成しファンクションの戻り値としている。SQL ServerではTABLE変数へ変換する。

# 用途

* 呼び出し元のSQLで、ファンクションをFROM句に指定して使用できるようになる。
* テーブル変数は更新時のみ、ロックとログのリソースを使用するため、リソースの節約になる。
* 数百行程度のレコードを格納する場合に使用する。

# SQL

## 変換前

```SQL
CREATE OR REPLACE TYPE NUMBER_LIST AS TABLE OF INTEGER;
CREATE OR REPLACE FUNCTION SAMPLE
(
    pStart INTEGER DEFAULT 0, 
    pEnd INTEGER DEFAULT 10
)
RETURN NUMBER_LIST
IS
    wk INTEGER;
BEGIN
    wk := pStart;
    WHILE wk <= pEnd LOOP 
        PIPE ROW(wk);
        wk := wk + 1;
    END LOOP;

    RETURN;
END;
SELECT * FROM SAMPLE();
```

## 変換後

```SQL
CREATE OR ALTER FUNCTION NUMBER_LIST (
    @pStart int = 0,
    @pEnd int = 10
) RETURNS @NUMBER_LIST TABLE (
    sequence_no int
)
AS
BEGIN
    DECLARE @wk int 

    SET @wk = @pStart
    WHILE @wk <= @pEnd
    BEGIN
        INSERT @NUMBER_LIST SELECT @wk
        SET @wk = @wk + 1
    END
    RETURN
END;
GO
SELECT * FROM NUMBER_LIST(default, default);
```
