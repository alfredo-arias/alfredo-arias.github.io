---
layout: post
title: "Aplicando un  Filtro a sp_MSforeachDB"
date: 2015-11-05 14:25:00 -0400
categories: MSSQL
author: "Alfredo Arias"
---

## Problema:

Cuando se necesita trabajar sobre multiples bases de datos en una misma instnacia, a veces, se necesita ejecutar una consulta en cada una de estas, sí estamos utilizando el procedimiento almacenado no documentado sp_MSforeachDB, nos encontraremos en ciertos casos en la obligación de excluir o filtrar algunas de estas bases datos, por ejemplos las bases de datos del sistema (master, tempdb, model y msdb) o cualquier otra base de datos que por una razón u otra no aplique para el caso.

## Solución:

Para poder filtrar las bases de datos que no queremos que se procesen, podemos utilizar la siguiente sintaxis:

```sql
USE MASTER;
GO
EXEC sp_MSforeachdb 'IF ''?''  NOT IN (''master'',''tempdb'',''model'',''msdb'')
BEGIN
       SELECT name, physical_name,state,size
       FROM ?.sys.database_files
END';
GO
```

## Referencias:

- [SQL Server: Applying Filter on sp_MSforeachDB](http://www.codeproject.com/Articles/459536/SQL-Server-Applying-Filter-on-sp-MSforeachDB)