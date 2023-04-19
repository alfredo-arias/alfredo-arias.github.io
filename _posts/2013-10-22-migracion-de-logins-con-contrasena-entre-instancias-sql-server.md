---
layout: post
title: "Migración de Logins con Contraseña entre Instancias SQL Server"
date: 2013-10-22 20:24:00 -0400
categories: MSSQL
author: "Alfredo Arias"
---

## Problema:

En nuestro día a día como DBAs, muchas veces nos encontramos con la necesidad de transferir todas o algunas de las bases de datos de un servidor a otro, ya sea por alguna falla a nivel de hardware o actualización de equipos o por una simple re-estructuración a nivel de infraestructura, sea cual sea el motivo de este cambio trae consigo muchos inconvenientes tanto administrativos como de seguridad, ya que al cambiar de servidor perderemos los logins asociados a nuestros usuarios en las bases de datos debido a que estos se encuentran relacionados a través del Security Identifier (SID) del login el cual se perderá si simplemente creamos otros logins en el servidor/instancia destino, sin contar que también perderemos las contraseñas de todos los accesos, recrear todos los logins, reasignar contraseñas, actualizar los archivos de configuración de todas las aplicaciones que accedían a estas bases de datos generaría una carga administrativa y un tiempo fuera de línea para nuestras aplicaciones, servicios y usuarios demasiado alta y a la vez inaceptable.

## Solución:
1.- Conectate a la instancia de origen con SQL Server Management Studio con el IDE de su preferencia.

2.- Abre una nueva ventana del Query Editor y corre el siguiente script. Este script crea dos procedimientos almacenados en la base de datos `master` puedes utilizar una base de datos cualquiera no necesariamente `master`. Estos procedimientos son `sp_hexadecimal` y `sp_help_revlogin`. El primero convierte un string a hexadecimal y el segundo crea un login con la contraseña en hexadecimal.

```sql
USE master
GO

IF OBJECT_ID ('sp_hexadecimal') IS NOT NULL
  DROP PROCEDURE sp_hexadecimal
GO

CREATE PROCEDURE sp_hexadecimal
    @binvalue VARBINARY(256),
    @hexvalue VARCHAR (514) OUTPUT
AS

DECLARE @charvalue VARCHAR (514)
DECLARE @i INT
DECLARE @length INT
DECLARE @hexstring CHAR(16)

SELECT @charvalue = '0x'
SELECT @i = 1
SELECT @length = DATALENGTH (@binvalue)
SELECT @hexstring = '0123456789ABCDEF'

WHILE (@i <= @length)
BEGIN
    DECLARE @tempint INT
    DECLARE @firstint INT
    DECLARE @secondint INT
    SELECT @tempint = CONVERT(INT, SUBSTRING(@binvalue,@i,1))
    SELECT @firstint = FLOOR(@tempint/16)
    SELECT @secondint = @tempint - (@firstint*16)
    SELECT @charvalue = @charvalue +
        SUBSTRING(@hexstring, @firstint+1, 1) +
        SUBSTRING(@hexstring, @secondint+1, 1)
    SELECT @i = @i + 1
END

SELECT @hexvalue = @charvalue
GO

IF OBJECT_ID ('sp_help_revlogin') IS NOT NULL
  DROP PROCEDURE sp_help_revlogin
GO
CREATE PROCEDURE sp_help_revlogin @login_name SYSNAME = NULL AS
DECLARE @name SYSNAME
DECLARE @type VARCHAR (1)
DECLARE @hasaccess INT
DECLARE @denylogin INT
DECLARE @is_disabled INT
DECLARE @PWD_varbinary  VARBINARY (256)
DECLARE @PWD_string  VARCHAR (514)
DECLARE @SID_varbinary VARBINARY (85)
DECLARE @SID_string VARCHAR (514)
DECLARE @tmpstr  VARCHAR (1024)
DECLARE @is_policy_checked VARCHAR (3)
DECLARE @is_expiration_checked VARCHAR (3)

DECLARE @defaultdb SYSNAME
  
IF (@login_name IS NULL)
  DECLARE login_curs CURSOR FOR

      SELECT p.sid, p.name, p.TYPE, p.is_disabled, p.default_database_name, l.hasaccess, l.denylogin FROM
sys.server_principals p LEFT JOIN sys.syslogins l
      ON ( l.name = p.name ) WHERE p.TYPE IN ( 'S', 'G', 'U' ) AND p.name <> 'sa'
ELSE
  DECLARE login_curs CURSOR FOR


      SELECT p.sid, p.name, p.TYPE, p.is_disabled, p.default_database_name, l.hasaccess, l.denylogin FROM
sys.server_principals p LEFT JOIN sys.syslogins l
      ON ( l.name = p.name ) WHERE p.TYPE IN ( 'S', 'G', 'U' ) AND p.name = @login_name
OPEN login_curs

FETCH NEXT FROM login_curs INTO @SID_varbinary, @name, @type, @is_disabled, @defaultdb, @hasaccess, @denylogin
IF (@@fetch_status = -1)
BEGIN
  PRINT 'No login(s) found.'
  CLOSE login_curs
  DEALLOCATE login_curs
  RETURN -1
END
SET @tmpstr = '/* sp_help_revlogin script '
PRINT @tmpstr
SET @tmpstr = '** Generated ' + CONVERT (VARCHAR, GETDATE()) + ' on ' + @@SERVERNAME + ' */'
PRINT @tmpstr
PRINT ''
WHILE (@@fetch_status <> -1)
BEGIN
  IF (@@fetch_status <> -2)
  BEGIN
    PRINT ''
    SET @tmpstr = '-- Login: ' + @name
    PRINT @tmpstr
    IF (@type IN ( 'G', 'U'))
    BEGIN -- NT authenticated account/group

      SET @tmpstr = 'CREATE LOGIN ' + QUOTENAME( @name ) + ' FROM WINDOWS WITH DEFAULT_DATABASE = [' + @defaultdb + ']'
    END
    ELSE BEGIN -- SQL Server authentication
        -- obtain password and sid
            SET @PWD_varbinary = CAST( LOGINPROPERTY( @name, 'PasswordHash' ) AS VARBINARY (256) )
        EXEC sp_hexadecimal @PWD_varbinary, @PWD_string OUT
        EXEC sp_hexadecimal @SID_varbinary,@SID_string OUT
  
        -- obtain password policy state
        SELECT @is_policy_checked = CASE is_policy_checked WHEN 1 THEN 'ON' WHEN 0 THEN 'OFF' ELSE NULL END FROM sys.sql_logins WHERE name = @name
        SELECT @is_expiration_checked = CASE is_expiration_checked WHEN 1 THEN 'ON' WHEN 0 THEN 'OFF' ELSE NULL END FROM sys.sql_logins WHERE name = @name
  
            SET @tmpstr = 'CREATE LOGIN ' + QUOTENAME( @name ) + ' WITH PASSWORD = ' + @PWD_string + ' HASHED, SID = ' + @SID_string + ', DEFAULT_DATABASE = [' + @defaultdb + ']'

        IF ( @is_policy_checked IS NOT NULL )
        BEGIN
          SET @tmpstr = @tmpstr + ', CHECK_POLICY = ' + @is_policy_checked
        END
        IF ( @is_expiration_checked IS NOT NULL )
        BEGIN
          SET @tmpstr = @tmpstr + ', CHECK_EXPIRATION = ' + @is_expiration_checked
        END
    END
    IF (@denylogin = 1)
    BEGIN -- login is denied access
      SET @tmpstr = @tmpstr + '; DENY CONNECT SQL TO ' + QUOTENAME( @name )
    END
    ELSE IF (@hasaccess = 0)
    BEGIN -- login exists but does not have access
      SET @tmpstr = @tmpstr + '; REVOKE CONNECT SQL TO ' + QUOTENAME( @name )
    END
    IF (@is_disabled = 1)
    BEGIN -- login is disabled
      SET @tmpstr = @tmpstr + '; ALTER LOGIN ' + QUOTENAME( @name ) + ' DISABLE'
    END
    PRINT @tmpstr
  END

  FETCH NEXT FROM login_curs INTO @SID_varbinary, @name, @type, @is_disabled, @defaultdb, @hasaccess, @denylogin
   END
CLOSE login_curs
DEALLOCATE login_curs
RETURN 0
GO
```

3.- Ejecute el siguiente comando para obtener el script de creación de los usuarios:


```sql
EXEC sp_help_revlogin;
GO
```

El resultado de ejecutar el procedimiento almacenado `sp_help_revlogin` es nuestro script de creación de logins, el cual nos servirá para crear en nuestro servidor/instancia destino los accesos o logins con los Security Identifiers (SID) y contraseñas originales, y por lo tanto nuestros logins y usuarios serán identicos a los de nuestro servidor/instancia original sin necesidad de re-mapear los usuarios a los accesos (logins), ni tampoco tendremos la necesidad de reasignar contraseñas, ni actualizar los archivos de configuración de nuestras aplicaciones.


4.- En el servidor destino, inicia SQL Server Management Studio, y conectate a la instancia de SQL Server destino (a la cual deseamos mover las bases de datos y logins con Security Identifiers (SID) y Contraseñas.

5.- Ejecute el script de creación de logins en el servidor destino, y verifique que los logins se hayan creado correctamente.

## Referencias:

- [http://msdn.microsoft.com/es-es/library/ms178009.aspx](http://msdn.microsoft.com/es-es/library/ms178009.aspx)