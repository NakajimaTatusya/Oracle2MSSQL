# 概要

OracleのSQLCODEを、SQL ServerのERROR_NUMBER()へ変換する。

# 用途

* 各ファンクション、プロシージャで、成功時、エラー発生時の情報をログテーブルに書き込む。

# SQL

## 変換前

```SQL
--エクセプションが発生しキャッチされたところでは
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        ログ出力プロシージャ名(
            ストアドプロシージャファンクション名, 
            処理開始日時, 
            システム日時, 
            SQLCODE, 
            エラーメッセージ);
        COMMIT;
```

```SQL
--通常ブロックでのログ出力では
ログ出力プロシージャ名(
    ストアドプロシージャファンクション名, 
    処理開始日時, 
    システム日時, 
    SQLCODE, 
    'SUCCESS');
```

## 変換後

```SQL
--エクセプションが発生しキャッチされたところでは
～
BEGIN CATCH
    BEGIN
        IF @@TRANCOUNT > 0
            ROLLBACK WORK

        EXECUTE ログ出力プロシージャ名 
        @処理名 = ストアドプロシージャファンクション名, 
        @処理開始日時 = 処理開始日時, 
        @処理終了日時 = SYSDATETIME(), 
        @状態コード = ERROR_NUMBER(), 
        @メッセージ = ERROR_MESSAGE()

        IF @@TRANCOUNT > 0
            COMMIT WORK
    END
END CATCH
```

```SQL
--通常ブロックでのログ出力では
EXECUTE ログ出力プロシージャ名 
    @処理名 = ストアドプロシージャファンクション名, 
    @処理開始日時 = 処理開始日時, 
    @処理終了日時 = SYSDATETIME(), 
    @状態コード = 0, 
    @メッセージ = 'SUCCESS'
```

#　補足事項

通常のEXCEPTIONブロック内で使用されている場合は、正常に変換されているがssma_oracleを使用しているのでSQL Serverの機能を使用するように変更する。  
ERROR_NUMBER()の戻り値の型はint。  
ERROR_NUMBER()は、直前にエラーが無い場合は、NULLになる。  
処理が成功した場合、LOG_STATUS列へは、「0」を規定値として設定する。
