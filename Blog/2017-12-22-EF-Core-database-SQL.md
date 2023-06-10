---
layout: post
title: "Generate EF Core model from database using SQL script"
date: 2017-12-22
tags: dotnet sql ef
categories: programming
---
It is possible to generate EF Core model from existing database without some of tables. There is very simple and small SQL query which generated a content of C# file.

```sql
USE <<MyDatabase>>

DECLARE @classNames VARCHAR(MAX);
SET @classNames = 'TableName1,TableName2,TableName3';

DECLARE @schemaName NVARCHAR(MAX), @className NVARCHAR(MAX), @tableName NVARCHAR(MAX);;
SET @schemaName = 'dbo'
DECLARE @Result VARCHAR(MAX);

PRINT '// generated by script'
PRINT 'using Microsoft.EntityFrameworkCore;'
PRINT 'using System;'
PRINT 'using System.Collections.Generic;'
PRINT 'using System.ComponentModel.DataAnnotations;'
PRINT 'using System.ComponentModel.DataAnnotations.Schema;'
PRINT 'using System.Configuration;'

PRINT 'namespace Model.'+ DB_NAME() 
PRINT '{'
PRINT '    public class '+ DB_NAME() +'Context : DbContext {'

-- adding list of all tables where identity columns

SET @Result = ''
SELECT @Result = CASE WHEN @Result = '' THEN '"'+tbl.name+'"' ELSE @Result + ', "' + tbl.name + '"' END
FROM sys.columns cols
JOIN sys.tables tbl ON cols.object_id = tbl.object_id
WHERE tbl.name in (select DISTINCT [Value] from STRING_SPLIT(@classNames, ',')) AND cols.is_identity = 1

PRINT '        public List<string> TablesWithIdentity = new List<string>() {' + @Result + '};'

PRINT '        public '+ DB_NAME() +'Context() { }'
PRINT '        public '+ DB_NAME() +'Context(DbContextOptions<'+ DB_NAME() +'Context> options): base(options) { }'
PRINT '        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)'
PRINT '        {'
PRINT '            if (!optionsBuilder.IsConfigured)'
PRINT '                optionsBuilder.UseSqlServer(ConfigurationManager.ConnectionStrings["'+ DB_NAME() +'"].ConnectionString);'
PRINT '        }'

-- adding DbSets for each table
DECLARE CLASS_NAME CURSOR LOCAL FOR
SELECT DISTINCT [VALUE]
FROM STRING_SPLIT(@classNames, ',')
OPEN CLASS_NAME
FETCH NEXT FROM CLASS_NAME INTO @className
WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT '        public DbSet<'+@className+'> '+@className+'s { get; set; }'
    
    FETCH NEXT FROM CLASS_NAME INTO @className
END
CLOSE CLASS_NAME
DEALLOCATE CLASS_NAME

-- adding primary key for each table

PRINT '        protected override void OnModelCreating(ModelBuilder modelBuilder) {'
DECLARE CLASS_NAME CURSOR LOCAL FOR
SELECT DISTINCT [VALUE]
FROM STRING_SPLIT(@classNames, ',')
OPEN CLASS_NAME
FETCH NEXT FROM CLASS_NAME INTO @className
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @tableName = UPPER(@className)
    SET @Result = NULL
    SELECT @Result = CASE WHEN @Result IS NULL THEN 'x.' + cols.name ELSE @Result + ', x.' + cols.name END
    FROM sys.columns cols
    LEFT JOIN (
        SELECT i.is_primary_key, ic.object_id, ic.column_id
        FROM sys.index_columns ic
        LEFT JOIN sys.indexes i on ic.object_id = i.object_id and ic.index_id = i.index_id
        WHERE i.is_primary_key = 1) AS PRIMARYKEY ON PRIMARYKEY.column_id = cols.column_id AND PRIMARYKEY.object_id = cols.object_id
    JOIN sys.tables tbl ON cols.object_id = tbl.object_id
    WHERE tbl.name = @tableName AND PRIMARYKEY.is_primary_key = 1

    PRINT '            modelBuilder.Entity<'+@className+'>().HasKey(x => new {'+@Result+'});'
    IF @Result = 'x.ID' OR @Result = 'x.Id' OR @Result = 'x.id'
        PRINT '            modelBuilder.Entity<'+@className+'>().Property(x => '+@Result+').ValueGeneratedNever();'
    
    FETCH NEXT FROM CLASS_NAME INTO @className
END
CLOSE CLASS_NAME
DEALLOCATE CLASS_NAME
PRINT '        }'
PRINT '    }'
PRINT ''

-- start adding POCO objects for each table
DECLARE CLASS_NAME CURSOR LOCAL FOR
SELECT DISTINCT [VALUE]
FROM STRING_SPLIT(@classNames, ',')
OPEN CLASS_NAME
FETCH NEXT FROM CLASS_NAME INTO @className
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @tableName = UPPER(@className)
    
    DECLARE tableColumns CURSOR LOCAL FOR
    SELECT DISTINCT cols.name, cols.system_type_id, cols.is_nullable
    FROM sys.columns cols
    JOIN sys.tables tbl ON cols.object_id = tbl.object_id
    WHERE tbl.name = @tableName

    PRINT '    [Table("'+@tableName+'")]' 
    PRINT '    public partial class ' + @className
    PRINT '    {'
 
    OPEN tableColumns
    DECLARE @name NVARCHAR(MAX), @typeId INT, @isNullable BIT, @typeName NVARCHAR(MAX)
    FETCH NEXT FROM tableColumns INTO @name, @typeId, @isNullable
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @typeName =
        CASE @typeId
            WHEN 36 THEN 'Guid'
            WHEN 52 THEN 'int'
            WHEN 56 THEN 'int'
            WHEN 61 THEN 'DateTime'
            WHEN 104 THEN 'bool'
            WHEN 108 THEN 'decimal'
            WHEN 127 THEN 'long'
            WHEN 167 THEN 'string'
            WHEN 175 THEN 'string'
            WHEN 231 THEN 'string'
            WHEN 239 THEN 'string'
            WHEN 241 THEN 'XElement'
            ELSE 'TODO(' + CAST(@typeId AS NVARCHAR) + ')'
        END;
        IF @isNullable = 1 AND @typeId != 231 AND @typeId != 239 AND @typeId != 241 AND @typeId != 167 AND @typeId != 175
            SET @typeName = @typeName + '?'
        PRINT '        public ' + @typeName + ' ' + @name + ' { get; set; }'
        FETCH NEXT FROM tableColumns INTO @name, @typeId, @isNullable
    END

    PRINT '    }'
    PRINT ''
 
    CLOSE tableColumns
    DEALLOCATE tableColumns
    FETCH NEXT FROM CLASS_NAME INTO @className
END
CLOSE CLASS_NAME
DEALLOCATE CLASS_NAME

PRINT '}'
```

Unfortunately it is not generating any foreign keys for tables. So you have to add them using partial class if needed.
So current sample will generate a model with 3 entities.

Thanks.