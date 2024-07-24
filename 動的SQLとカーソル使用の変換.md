# 概要

Oracleで動的SQLとカーソルを用いた処理を、SQL Serverの「EXECUTE sp_executesql」へ変換する。

# 用途

* パフォーマンス向上のために、バインド変数を使用する。
* SQLに変数を文字列結合している個所がある場合は、バインド変数を使用する。
* ファンクション、プロシージャの引数で、検索条件が分岐する場合など、SQL構文自体が条件式によって構造を変更する必要がある場合に使用する。

# 基本的な動的SQL

## SQL

### 変換前

```SQL
query := 'SELECT MAX(ID) AS MAX_ID FROM TBL_EXAMPLE WHERE CAPTION = :cap';

EXECUTE IMMEDIATE query INTO max_id USING caption;
```

### 変換後

```SQL
DELETE FROM TBL_EXAMPLE;
INSERT INTO TBL_EXAMPLE VALUES (1, 'aaa');
INSERT INTO TBL_EXAMPLE VALUES (2, 'aaa');
INSERT INTO TBL_EXAMPLE VALUES (3, 'bbb');
INSERT INTO TBL_EXAMPLE VALUES (4, 'bbb');
INSERT INTO TBL_EXAMPLE VALUES (5, 'bbb');

DECLARE @result INT, @query NVARCHAR(max), @defineparams NVARCHAR(max);
SET @query = N'SELECT @result_out = MAX(ID) FROM TBL_EXAMPLE WHERE CAPTION = @condition';
SET @defineparams = N'@condition NVARCHAR(20), @result_out INT OUTPUT';

EXECUTE sp_executesql 
    @query,
    @defineparams, 
    @condition = 'bbb',
    @result_out = @result OUTPUT;

SELECT @result;
```

```SQL
DELETE FROM TBL_EXAMPLE;
INSERT INTO TBL_EXAMPLE VALUES (1, 'aaa');
INSERT INTO TBL_EXAMPLE VALUES (2, 'aaa');
INSERT INTO TBL_EXAMPLE VALUES (3, 'bbb');
INSERT INTO TBL_EXAMPLE VALUES (4, 'bbb');
INSERT INTO TBL_EXAMPLE VALUES (5, 'bbb');

SET @query = N'INSERT INTO TBL_EXAMPLE (ID, CAPTION) VALUES (@col1, @col2)';
SET @defineparams = N'@col1 INT, @col2 NVARCHAR(20)';

EXECUTE sp_executesql
    @query,
    @defineparams,
    @col1 = 6,
    @col2 = 'ccc';

SELECT * FROM TBL_EXAMPLE;
```

# 動的SQLとカーソル

## SQL

### 変換前

```SQL
～
IS
    TYPE recExample IS RECORD (ID NUMBER, CAPTION VARCHAR2);
    TYPE tableExample IS TABLE OF recExample;
    wkExample tableExample;
    TYPE typData IS REF CURSOR;
    crData typData;
    condition VARCHAR2 := 'bbb';
    query VARCHAR2;
～
query := 'SELECT ID, CAPTION FROM TBL_EXAMPLE WHERE CAPTION = :cap';

OPEN crData FOR query USING condition;

LOOP
    FETCH crData INTO wkExample;
    EXIT WHEN crData%NOTFOUND;
    ～
END LOOP;

CLOSE crData;
```

### 変換後


```SQL
DELETE FROM TBL_EXAMPLE;
INSERT INTO TBL_EXAMPLE VALUES (1, 'aaa');
INSERT INTO TBL_EXAMPLE VALUES (2, 'aaa');
INSERT INTO TBL_EXAMPLE VALUES (3, 'bbb');
INSERT INTO TBL_EXAMPLE VALUES (4, 'bbb');
INSERT INTO TBL_EXAMPLE VALUES (5, 'bbb');

DECLARE @DataList TABLE
(
    ID INT NOT NULL,
    CAPTION NVARCHAR(20),
    INDEX "uindex1" UNIQUE CLUSTERED(ID)
);
DECLARE @query NVARCHAR(max), @defineparams NVARCHAR(max);
DECLARE @result_id INT, @result_cap VARCHAR(20);

SET @query = N'DECLARE CursorSample CURSOR FOR SELECT ID, CAPTION FROM TBL_EXAMPLE WHERE CAPTION = @condition';
SET @defineparams = N'@condition NVARCHAR(20)';

EXECUTE sp_executesql @query, @defineparams, @condition = 'bbb';

OPEN CursorSample;

WHILE 1 = 1
BEGIN
    FETCH NEXT FROM CursorSample INTO @result_id, @result_cap
    IF  @@FETCH_STATUS <> 0
    BEGIN
        BREAK;
    END

    INSERT @DataList SELECT @result_id, @result_cap
END;

CLOSE CursorSample;
DEALLOCATE CursorSample;

SELECT * FROM @DataList;
```

```SQL
--以下は、Oracle PL/SQLで、テーブル型変数.Firstやテーブル型変数[index]結果セットにアクセスしている個所をSQL Serverにアレンジした。
--一意列をIDENTITY(0,1)で自動採番
DECLARE @DataList TABLE (
    SEQ INT NOT NULL IDENTITY(0,1),
    ID INT NOT NULL,
    CAPTION NVARCHAR(20),
    INDEX "uindex1" UNIQUE CLUSTERED(ID)
);
DECLARE @query NVARCHAR(max), @defineparams NVARCHAR(max);
DECLARE @result_id INT, @result_cap VARCHAR(20);

SET @query = N'DECLARE CursorSample CURSOR FOR SELECT ID, CAPTION FROM TBL_EXAMPLE WHERE CAPTION = @condition';
SET @defineparams = N'@condition NVARCHAR(20)';

EXECUTE sp_executesql @query, @defineparams, @condition = 'bbb';

OPEN CursorSample;

WHILE 1 = 1
BEGIN
    FETCH NEXT FROM CursorSample INTO @result_id, @result_cap
    IF  @@FETCH_STATUS <> 0
    BEGIN
        BREAK;
    END

    INSERT @DataList SELECT @result_id, @result_cap
END;

CLOSE CursorSample;
DEALLOCATE CursorSample;

DECLARE @identify INT;
DECLARE @title NVARCHAR(20);

SELECT * FROM @DataList;

--First
SELECT TOP 1 @identify = ID, @title = CAPTION FROM @DataList ORDER BY ID ASC;
SELECT @identify, @title;

--Table変数[インデックス]でアクセスする
SELECT @identify = ID, @title = CAPTION FROM @DataList WHERE SEQ = 1;
SELECT @identify, @title;
```

# SQL文が条件により変化する動的SQL

## SQL

### 変換前

```SQL
query := 'SELECT * FROM TBL_EXAMPLE2 WHERE '
IF flag = '1' THEN
    query := query || 'SCHOOL_YEAR = :sy';
ELSE
    query := query || 'SCHOOL_YEAR = :sy AND CLASS = :cl';
END IF;
```

### 変換後

```SQL
DELETE FROM TBL_EXAMPLE2;
INSERT INTO TBL_EXAMPLE2 VALUES (1, '1年1組', '1', '1');
INSERT INTO TBL_EXAMPLE2 VALUES (2, '1年2組', '1', '2');
INSERT INTO TBL_EXAMPLE2 VALUES (3, '1年3組', '1', '3');
INSERT INTO TBL_EXAMPLE2 VALUES (4, '2年1組', '2', '1');
INSERT INTO TBL_EXAMPLE2 VALUES (5, '2年2組', '2', '2');
INSERT INTO TBL_EXAMPLE2 VALUES (6, '2年3組', '2', '3');

DECLARE @flag CHAR(1), @p1 CHAR(1), @p2 CHAR(1);
DECLARE @query NVARCHAR(max), @defineparams NVARCHAR(max);
DECLARE @result_id INT, @result_cap NVARCHAR(20), @result_sc CHAR(1), @result_cl CHAR(1);

SET @flag = '1';

SET @query = N'DECLARE CursorSample CURSOR FOR SELECT * FROM TBL_EXAMPLE2 WHERE ';
SET @defineparams = N'@sy char(1), @cl char(1)';

IF @flag = '1'
BEGIN
    SET @query = CONCAT(@query, 'SCHOOL_YEAR = @sy');
    SET @p1 = '1';
END;
ELSE
BEGIN
    SET @query = CONCAT(@query, 'SCHOOL_YEAR = @sy AND CLASS = @cl');
    SET @p1 = '2';
    SET @p2 = '2';
END;

EXECUTE sp_executesql
    @query,
    @defineparams,
    @p1,
    @p2;

OPEN CursorSample;

WHILE 1 = 1
BEGIN
    FETCH NEXT FROM CursorSample INTO @result_id, @result_cap, @result_sc, @result_cl
    IF  @@FETCH_STATUS <> 0
    BEGIN
        BREAK;
    END

    PRINT CONCAT(@result_id, ' | ', @result_cap, ' | ', @result_sc, ' | ', @result_cl)
END;

CLOSE CursorSample;
DEALLOCATE CursorSample;
```

# 補足事項

* 「SQL文が条件により変化する動的SQL」の処理は制御結合になっている。既存の処理はこのような制御結合を行っているが、本来望ましくない。

# 関連エラー

* エラーコード：O2SS0013 への対応
* エラーコード：O2SS0157 への対応
* エラーコード：O2SS0422 への対応
* 警告コード：O2SS0423 への対応
