# 概要

SQL ServerにROWIDは存在しないので、主キーを絞り込み条件とする。

# 用途

* UPDATE文の更新対象絞り込み条件指定

# SQL

## 変換前

```SQL
OPEN cursor FOR SELECT ROWID FROM TBL_EXAMPLE;

FETCH cursor BULK COLLECT INTO typeTable;

CLOSE cursor;

WHILE i IS NOT NULL LOOP
    UPDATE TBL_EXAMPLE
        SET
            CAPTION = 'AAAAAAA'
        WHERE ROWID = typeTable(i).ROWID;
END LOOP;
```

## 変換後

```SQL
DELETE FROM TBL_EXAMPLE;
INSERT INTO TBL_EXAMPLE VALUES (1, 'aaaaaa');
INSERT INTO TBL_EXAMPLE VALUES (2, 'bbbbbb');
INSERT INTO TBL_EXAMPLE VALUES (3, 'cccccc');

DECLARE @targetid INT;
DECLARE CursorSample CURSOR FORWARD_ONLY STATIC READ_ONLY
    FOR SELECT ID FROM TBL_EXAMPLE;

OPEN CursorSample;

WHILE 1 = 1
BEGIN
    FETCH NEXT FROM CursorSample INTO @targetid;
    IF @@FETCH_STATUS <> 0
    BEGIN
        BREAK;
    END;
    
    UPDATE TBL_EXAMPLE SET CAPTION = 'CAPTION' WHERE ID = @targetid;
END;

CLOSE CursorSample;
DEALLOCATE CursorSample;
```
