# 概要

OracleのINSERT文VALUES句に、サブクエリが使われていた場合変換エラーを出力する。

# 用途

* INSERT文のVALUES句へサブクエリを指定する。

# SQL

## 変換前

```SQL
variable1 NUMBER := 1
INSERT INTO RESULT_TABLE (ID, SITUATION, DATEADDED) VALUES
(
    variable1,
    (SELECT SUBSTR(StaffName, 0, 10) FROM Employee WHERE StaffNum = '00001'),
    SYSTIMESTAMP
);
```

## 変換後

```SQL
DELETE FROM Employee;
INSERT INTO Employee VALUES ('00001', 'TEMP STAFF1', 'A');
INSERT INTO Employee VALUES ('00002', 'TEMP STAFF2', 'A');
INSERT INTO Employee VALUES ('00003', 'TEMP STAFF3', 'A');
INSERT INTO Employee VALUES ('00004', 'TEMP STAFF4', 'B');
INSERT INTO Employee VALUES ('00005', 'TEMP STAFF5', 'B');

DELETE FROM RESULT_TABLE;
DECLARE @variable1 INT = 1;
INSERT INTO RESULT_TABLE (ID, SITUATION, DATEADDED) VALUES
(
    @variable1,
    (SELECT SUBSTRING(StaffName, 1, 10) FROM Employee WHERE StaffNum = '00002'),
    SYSDATETIME()
)
```
