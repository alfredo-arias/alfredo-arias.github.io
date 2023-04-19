---
layout: post
title: "IF EXISTS, Verificar Si una Tabla Temporal Existe SQL Server"
date: 2013-04-04 10:29:00 -0400
categories: MSSQL
author: "Alfredo Arias"
---

## Problema:

¿Como verifico si una tabla temporal existe tanto a nivel local como a nivel global en SQL Server?

## Solución:

Para verificar si una tabla temporal existe en SQL Server, podemos utilizar la siguiente sintaxis:

```sql
IF OBJECT_ID('tempdb..#temp') IS NOT NULL

 -- Cambiamos el contexto al de nuestra base de datos de ejemplo  
 USE Norhtwind  
 GO  
 -- Creamos una tabla temporal dentro de esta  
 CREATE TABLE #temp(id INT)  
 -- Verificamos si nuestra tabla existe  
 IF OBJECT_ID('tempdb..#temp') IS NOT NULL  
   BEGIN  
    PRINT '¡#temp existe!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡#temp does not exist!'  
   END  
 -- Otra forma de verificar si esta tabla existe es   
 -- utilizando un segundo parametro opcional no documentado  
 IF OBJECT_ID('tempdb..#temp','u') IS NOT NULL  
   BEGIN  
    PRINT '¡#temp existe!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡#temp no existe!'  
   END  
 -- No hagas esto porque esto verificara la base de datos  
 -- local y por lo tanto retornara no existe   
 IF OBJECT_ID('tempdb..#temp','local') IS NOT NULL  
   BEGIN  
    PRINT '¡#temp existe!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡#temp no existe!'  
   END  
 -- A menos que cambiemos el contexto de la   
 -- base de datos a algo como esto  
 USE tempdb  
 GO  
 -- Ahora la tabla existe de nuevo  
 IF OBJECT_ID('tempdb..#temp','local') IS NOT NULL  
   BEGIN  
    PRINT '¡#temp exists!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡#temp no existe!'  
   END  
 -- Volvamos a Norhtwind de nuevo  
 USE Norhtwind  
 GO  
 -- Verificamos si la tabla temporal existe  
 IF OBJECT_ID('tempdb..#temp') IS NOT NULL  
   BEGIN  
    PRINT '¡#temp exists!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡#temp does not exist!'  
   END
```

 Ahora abre una nueva ventana del Analizador de Consultas (Query Analyzer) (CTRL + N) y ejecuta este c&oacute;digo de nuevo.  

```sql
 -- Verificar si existe  
 IF OBJECT_ID('tempdb..#temp') IS NOT NULL  
   BEGIN  
    PRINT '¡#temp existe!'  
   END  
 ELSE   
   BEGIN  
    PRINT '¡#temp no existe!'  
   END
```

La tabla no existe y es correcto ya que esta es una tabla temporal de alcance local, no una tabla temporal de alcance global. Probemos ahora esta otra sentencia.  
 
```sql
 -- Crea una tabla temporal de alcance global   
 -- Note los 2 signos de número, esto crea la tabla a nivel global  
 CREATE TABLE ##temp(id INT)   
 -- Verifica si esta existe  
 IF OBJECT_ID('tempdb..##temp') IS NOT NULL  
   BEGIN  
    PRINT '¡##temp existe!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡##temp no existe!'  
 END  
 ```
 ¿Existe, correcto?  
 Ahora ejecutamos el mismo c&oacute;digo en una ventana nueva del Analizador de Consultas (Query Analyzer) (CTRL + N).  
 
```sql
 -- Verificamos si existe  
 IF OBJECT_ID('tempdb..##temp') IS NOT NULL  
   BEGIN  
    PRINT '¡##temp existe!'  
   END  
 ELSE  
   BEGIN  
    PRINT '¡##temp no existe!'  
   END
```  

 Y si esta esta vez existe ya que esta es una tabla de alcance global.  

## Referencias:

- [http://sqlservercodebook.blogspot.com/2008/03/check-if-temporary-table-exists.html](http://sqlservercodebook.blogspot.com/2008/03/check-if-temporary-table-exists.html)