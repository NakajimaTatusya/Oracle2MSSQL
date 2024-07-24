# 概要

* SQL Serverではカーソル毎のステータスを@@FETCH_STATUSで区別しないため、@@FETCH_STATUSのタイミングに注意する。

# 用途

* @@FETCH_STATUSでFETCHの可否判定を行う。

# SQL

## 変換前

```SQL
～
IS
    TYPE typCur IS REF CURSOR;
    cursor1 typCur;
    cursor2 typCur;
～
OPEN cursor1 FOR 'SELECT StaffNum FROM Employee';
LOOP
    FETCH cursor1 INTO record;
    EXIT WHEN curosr1%NOTFOUND;

    OPEN cursor2 FOR 'SELECT StaffName FROM Employee WHERE StaffNum = :variable' USING cursor1.StaffNum;
    FETCH cursor2 INTO record2;
    IF curosr2%NOTFOUND = FALSE THEN
        /* 処理 */
    END IF;

END LOOP;
CLOSE cursor1;
CLOSE cursor2;
～
```

## 変換後

```SQL
DROP PROCEDURE NESTED_CURSOR_DEMO
GO

CREATE OR ALTER PROCEDURE NESTED_CURSOR_DEMO
AS
BEGIN
    DECLARE @varID_OUTER CHAR(5), @varName NVARCHAR(20);

    DECLARE cur1 CURSOR LOCAL FOR SELECT StaffNum FROM Employee;
    OPEN cur1;
    WHILE 1=1
    BEGIN
        FETCH NEXT FROM cur1 INTO @varID_OUTER; /* FETCHと@@FETCH_STATUSをセットでなるべく使用する。 */
        IF  @@FETCH_STATUS <> 0
            BREAK;

        DECLARE cur2 CURSOR LOCAL FOR SELECT StaffName FROM Employee WHERE StaffNum = @varID_OUTER;
        OPEN cur2;
        FETCH NEXT FROM cur2 INTO @varName; /* FETCHと@@FETCH_STATUSをセットでなるべく使用する。 */
        IF @@FETCH_STATUS = 0
            SELECT @varName;
        CLOSE cur2;
        DEALLOCATE cur2;
    END;
    CLOSE cur1;
    DEALLOCATE cur1;
END
GO

EXEC NESTED_CURSOR_DEMO;
```

# 補足事項

* SQL ServerではカーソルのSELECTステートメントに記述されたバインド変数は、DECLARE時に評価されるため  
バインド変数の中身が変更される場合は、CLOSE、DELLOCATEを呼んだ後、再度DECLAREにてカーソルを定義する必要がある。
