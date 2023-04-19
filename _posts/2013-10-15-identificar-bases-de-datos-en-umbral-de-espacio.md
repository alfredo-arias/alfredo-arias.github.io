---
layout: post
title: "Identificar Bases de Datos Cerca del Umbral de Espacio en Disco"
date: 2013-10-15 21:37:00 -0400
categories: MSSQL
author: "Alfredo Arias"
---

## Problema:
En muchos servidores de SQL Server el tamaño máximo de los archivos, ya sea de datos o de log, puede estar restringido para asegurar que el espacio disponible en el servidor es adecuado. El problema con esto es que si tus archivos de datos o log se quedan sin espacio tendrás un error como el que se muestra a continuación y tus transacciones fallaran.

```
Msg 1105, Level 17, State 2, Line 1 Could not allocate space for object 'dbo.table1'
in database 'test' because the 'PRIMARY' filegroup is full. Create disk space 
by deleting unneeded files, dropping objects in the filegroup, adding 
additional files to the filegroup, or setting autogrowth on for existing
files in the filegroup.
```

## Solución:
La solución general que sugiero es un procedimiento almacenado llamado dbo.obtener_archivos_bd_cerca_maxsize. Este procedimiento toma un parámetro opcional para el porcentaje de espacio, si ningún parámetro es suministrado este tomará un valor por defecto de 10%. Luego, este verificar cada archivo en cada una de las bases de datos en el servidor, incluyendo las bases de datos del sistema.

Si ejecutas el procedimiento almacenado sin pararle el parámetro este buscara todos los archivos de base de datos, tanto de datos como log, que tengan disponible 10% o menos de espacio, solo para los archivos que tengan configurado un máximo de espacio.

Aquí está el procedimiento almacenado, el cual puede ser creado en master o en su base de datos de administración de poseer alguna.

```sql
CREATE PROCEDURE [dbo].[obtener_archivos_bd_cerca_maxsize]
     (@porcentaje_umbral_espacio DECIMAL (5,1) = 10.0)AS
BEGIN
    SET NOCOUNT ON
 
    -- Crea tabla temporal global
    CREATE TABLE ##todos_archivos_bd (
        [dbname] SYSNAME,
        [fileid] SMALLINT,
        [groupid] SMALLINT,
        [size] INT NOT NULL,
        [maxsize] INT NOT NULL,
        [growth] INT NOT NULL,
        [status] INT,
        [perf] INT,
        [name] SYSNAME NOT NULL,
        [filename] NVARCHAR(260) NOT NULL
    )
     -- Itera sobre todas las bases de datos y recopila información
     -- de la vista del sistema 'sysfiles' y la introduce en la tabla
     -- temporal gblogal 'all_db_files' usando el procedimiento
     -- almacenado no documentado 'sp_MsForEachDB'
    
    EXEC sp_MsForEachDB
            @command1 = 'USE [$]; INSERT INTO ##todos_archivos_bd SELECT DB_NAME(), * FROM SysFiles',
            @replacechar = '$'

    -- Salida de los resultados
     SELECT
          [dbname] AS [database_name],
          [name] AS [nombre_logico_archivo],
          [filename] AS [ruta_archivo_fisico],
          ROUND(size * CONVERT(FLOAT,8) / 1024,0) AS [tamano_actual_mb],
          ROUND(maxsize * CONVERT(FLOAT,8) / 1024,0) AS [tamano_max_restringido_mb],
          ROUND(maxsize * CONVERT(FLOAT,8) / 1024,0) - ROUND(size * CONVERT(FLOAT,8) / 1024,0) AS [espacio_restante_mb]
     FROM   ##todos_archivos_bd
     -- Omitir los archivos de base de datos que no tienen un
     -- tamaño máximo definido
     WHERE     maxsize > -1
     -- Encontrar los archivos de base de datos dentro del
     -- porcentaje establecido
     AND ([maxsize] - [size]) * 1.0 < 0.01 * @porcentaje_umbral_espacio * [maxsize]
     ORDER BY 6
    
     DROP TABLE ##todos_archivos_bd
     SET NOCOUNT OFF
END
GO
```
Este es un ejemplo de ejecución. Este muestra que ambos, tanto el archivo de datos como el de log están casi a su máximo de tamaño y que el archivo de datos tiene 3MB libres y el archivo de log tiene 4MB libres para la base de datos `test`.

![Ejemplo](/assets/images/82a3a064-353d-4afe-9622-f068872b08ff.png)

Como administradores de bases de datos deberíamos ejecutar este procedimiento semanalmente o inclusive diariamente de ser posible para buscar todos los archivos de base de datos que estén acercándose al límite  máximo de tamaño. Entonces dependerá de nosotros resolver el problema agregando más espacio al archivo o tomando las medidas correctivas de lugar.

Idealmente el espacio del disco no debería de ser un problema y no deberías de preocuparte por el tamaño máximo de los archivos, pero incluso con el bajo costo del espacio en disco aun muchos sistemas están limitado en este aspecto y como administradores de bases de datos debemos de hacer el máximo con los recursos que poseemos.


## Referencias:
- [http://msdn.microsoft.com/es-es/library/ms178009.aspx](http://msdn.microsoft.com/es-es/library/ms178009.aspx)