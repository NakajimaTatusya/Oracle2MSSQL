# 概要

Oracleでは「FOR UPDATE WAIT n」でロック待機時間を設定し、リトライによりデッドロックを回避してトランザクションを続行している。  
SQL Serverで同等の実装を行う方法を記述する。

# 用途

* デッドロックを検出する。

# SQL

## 変換前

```SQL
～
FOR i IN 1..3 LOOP
BEGIN
    SELECT StaffNum BULK COLLECT INTO tableVariable
    FROM Employee
    WHERE StaffNum BETWEEN '00001' AND '00003'
    FOR UPDATE WAIT 10;

    GOTO LABEL_RETRY;

    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            GOTO LABEL_RETRY;
        WHEN TIMEOUT_WAIT OR TIMEOUT_NOWAIT THEN
            IF i < 3 THEN
                NULL;
            ELSE
                ROLLBACK WORK;
                GOTO LABEL_ERROR;
            END IF;
        WHEN OTHERS THEN
            ROLLBACK WORK;
            GOTO LABEL_ERROR;
END
END LOOP;
<<LABEL_RETRY>>
～
<<LABEL_ERROR>>
～
```

## 変換後

```SQL
DROP PROCEDURE TRANSACTION_A
GO

CREATE OR ALTER PROCEDURE TRANSACTION_A
AS
BEGIN
    DECLARE @RetryCnt INT;
    SET @RetryCnt = 1;
    RETRY_LABEL:
    
    BEGIN TRANSACTION
    BEGIN TRY
        UPDATE Customer SET CUSTOMER_NAME = '富士山男' WHERE CUSTOMER_ID = 1;
        WAITFOR DELAY '00:00:05';
        UPDATE Shipping SET CUSTOMER_ID = 1 WHERE SHIPPING_ID = 1;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        PRINT 'ロールバックトランザクションA';
        ROLLBACK TRANSACTION;
        DECLARE @DoRetry BIT;
        DECLARE @ErrMsg VARCHAR(500);

        SET @DoRetry = 0;
        SET @ErrMsg = ERROR_MESSAGE();

        IF ERROR_NUMBER() = 1205    ---Deadlock error number
        BEGIN
            SET @DoRetry = 1;
        END

        IF @DoRetry = 1
        BEGIN
            SET @RetryCnt += 1;
            IF (@RetryCnt > 3)
            BEGIN
                RAISERROR(@ErrMsg, 18, 1);
            END
            ELSE
            BEGIN
                PRINT @ErrMsg;
                WAITFOR DELAY '00:00:05';
                GOTO RETRY_LABEL;
            END
        END
        ELSE
        BEGIN
            RAISERROR(@ErrMsg, 18, 1);
        END
    END CATCH
END
GO
```

```SQL
DROP PROCEDURE TRANSACTION_B
GO

CREATE OR ALTER PROCEDURE TRANSACTION_B
AS
BEGIN
    DECLARE @RetryCounter INT;
    SET @RetryCounter = 1;
    RETRY_LABEL: --label retry

    BEGIN TRANSACTION
    BEGIN TRY
        UPDATE Shipping SET SHIPPING_NAME = '注文B' WHERE SHIPPING_ID = 1;
        WAITFOR DELAY '00:00:05';
        UPDATE Customer SET CUSTOMER_NAME = '八ヶ岳巌' WHERE CUSTOMER_ID = 1;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        PRINT 'ロールバックトランザクションB';
        ROLLBACK TRANSACTION;
        DECLARE @DoRetry BIT;
        DECLARE @ErrorMessage VARCHAR(500);
  
        SET @DoRetry = 0;
        SET @ErrorMessage = ERROR_MESSAGE();

        IF ERROR_NUMBER() = 1205 ---Deadlock Error Number
        BEGIN
            SET @DoRetry = 1;
        END

        IF @DoRetry = 1
        BEGIN
            SET @RetryCounter += 1;
            IF (@RetryCounter > 3)
            BEGIN
                RAISERROR(@ErrorMessage, 18, 1);
            END
            ELSE
            BEGIN
                PRINT @ErrorMessage;
                WAITFOR DELAY '00:00:05';
                GOTO RETRY_LABEL;
            END
        END
        ELSE
        BEGIN
            RAISERROR(@ErrorMessage, 18, 1);
        END
    END CATCH
END
GO
```

```SQL
DELETE FROM Customer;
DELETE FROM Shipping;
INSERT INTO Customer VALUES (1, '顧客１');
INSERT INTO Customer VALUES (2, '顧客２');
INSERT INTO Customer VALUES (3, '顧客３');
INSERT INTO Shipping VALUES (1, 1, '注文１');
INSERT INTO Shipping VALUES (2, 2, '注文２');

--TRANSACTION_AとTRANSACTION_Bを同時に実行する
EXEC TRANSACTION_A;
EXEC TRANSACTION_B;
```


# 補足事項

* デッドロックが発生すると、SQL Serverは自動的に1つを選択してプロセスを中止し、もう一方のプロセスを続行できるようにしてデッドロックを終了します。  
SQL Serverが選択するプロセスは、一般にロールバックするのに最小限のオーバヘッド（ロールバックされるデータ量が少ないもの）を必要とするトランザクションです。
* 仕様上どちらか一方を優先させる場合は、1205エラーで即時ストアドを停止するようコードを変更してください。
