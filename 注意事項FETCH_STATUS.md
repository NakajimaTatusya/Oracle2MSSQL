# 概要

Oracleでは、FETCHしたカーソルの状態を評価して、ループ処理の終わりを判断している。  
SQL Serverでは、同じような実装だと1回余計にループするので、変換後のような実装を行う。

# SQL

## 変換前

```SQL
LOOP
    FETCH cursor INTO variable;
    EXIT WHEN cursor%NOTFOUND;

    --何らかの処理

END LOOP;
```

## 変換後

```SQL
WHILE 1 = 1
BEGIN
    FETCH NEXT FROM @cursorName INTO @variable;
    IF @@FETCH_STATUS <> 0
    BEGIN
        BREAK;
    END

    PRINT(@variable);
END;
```
