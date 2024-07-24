# 概要

Oracleストアドファンクション内部で更新処理を行っている場合は、SQL Serverストアドプロシージャへ変換する。  
呼び出し元へ値を返す場合は、OUTPUTパラメータを使用を基本とする。

# 用途

* 更新処理を行った後、結果を整数で返す場合は「RETURNコードを使った値の返却」を使用。
* 更新処理を行った後、結果をOUTPUTパラメータで返す場合は「OUTPUTパラメータを使った値の返却」を使用。
* 更新結果セットを返す場合は「結果セットを使用してデータを返す」を使用。

# SQL

## RETURNコードを使った値の返却

### 変換前

```SQL
CREATE OR REPLACE FUNCTION ORACLE_FUNCTION
    RETURN INTEGER 
IS
BEGIN
    INSERT INTO Employee VALUES ('00006', 'TEST', 'C');
    RETURN 0;
END
/
```

### 変換後

```SQL
CREATE OR ALTER PROCEDURE SQLSERVER_PROCEDURE_1
AS
BEGIN
    INSERT INTO Employee VALUES ('00006', 'TEST6', 'C');
    RETURN(0)
END
GO
```

## OUTPUTパラメータを使った値の返却

### 変換前

```SQL
CREATE OR REPLACE FUNCTION ORACLE_FUNCTION (
    pout OUT VARCHAR2
)
IS
BEGIN
    INSERT INTO Employee VALUES ('00006', 'TEST', 'C');
    pout := 0;
    RETURN;
END
/
```

### 変換後

```SQL
CREATE OR ALTER PROCEDURE SQLSERVER_PROCEDURE_2 (
    @return_value_argument int OUTPUT
)
AS
BEGIN
    INSERT INTO Employee VALUES ('00007', 'TEST7', 'C');
    SET @return_value_argument = 7;
    RETURN
END
GO
```

## 結果セットを使用してデータを返す

### 変換前

```SQL
CREATE OR REPLACE FUNCTION ORACLE_FUNCTION
    RETURN TYPE_Employee_TABLE PIPELINED 
IS
BEGIN
    FOR cursor IN (SELECT * FROM Employee WHERE StaffKind = 'C') LOOP
        PIPE ROW (
            cursor.StaffNum,
            cursor.StaffName,
            cursor.StaffKind
        )
    END LOOP;
    RETURN;
END
/
```

### 変換後

```SQL
CREATE OR ALTER PROCEDURE SQLSERVER_PROCEDURE_3
AS
BEGIN
    SELECT * FROM Employee WHERE StaffKind = 'C';
    RETURN
END
GO
```

## サンプルSQLの実行

```SQL
DELETE FROM Employee;
INSERT INTO Employee VALUES ('00001', 'TEMP STAFF1', 'A');
INSERT INTO Employee VALUES ('00002', 'TEMP STAFF2', 'A');
INSERT INTO Employee VALUES ('00003', 'TEMP STAFF3', 'A');
INSERT INTO Employee VALUES ('00004', 'TEMP STAFF4', 'B');
INSERT INTO Employee VALUES ('00005', 'TEMP STAFF5', 'B');

DECLARE @retVal int;
EXEC @retVal = SQLSERVER_PROCEDURE_1;
SELECT @retVal;

EXEC SQLSERVER_PROCEDURE_2 @retVal OUTPUT;
SELECT @retVal;

EXEC SQLSERVER_PROCEDURE_3;
```

# 補足事項

* SQL句で使用する場合は、RETURN(整数値)を使用するか、呼び出し側を修正する。
