---
title: mysql表名列名转小写
date: 2022-11-12 17:05:57
tags:
- 小技巧
---



记录一下，根据工作中项目交付要求，要将MySQL数据库中的表名和字段名中做一个规范，其中就有将表名和字段名统一做小写处理。

        废话不多说，直接上MySQL脚本：

批量修改数据库下的表名（大写改小写）：

```sql
SELECT
 concat(
	 'rename table  ' , TABLE_NAME , ' to ' , LOWER(TABLE_NAME) ,' ;' ) AS '修改脚本sql'
FROM
 information_schema.TABLES t 
WHERE
 TABLE_SCHEMA = '数据库名';
```

批量修改列名（大写改小写）：

```sql
SELECT
	concat(
		'alter table ',
		TABLE_NAME,
		' change column ',
		COLUMN_NAME,
		' ',
		LOWER( COLUMN_NAME ),
		' ',
		COLUMN_TYPE,
		' comment \'',
		TRIM(
			REPLACE (
				REPLACE ( REPLACE ( REPLACE ( COLUMN_COMMENT, ',', ':' ), '"', '' ), CHAR ( 10 ), '' ),
				CHAR ( 13 ),
				'' 
			)),
		'\'',
		' ',
	IF
		(
			COLUMN_DEFAULT IS NULL,
			'',
		concat( ' default \'', TRIM( COLUMN_DEFAULT ), '\'' )),
		';' 
) AS '修改脚本sql' 
FROM
	information_schema.COLUMNS t 
WHERE
	TABLE_SCHEMA = '数据库名';
```

如果是小写改大写，只需要将LOWER 修改为 UCASE即可。

运行脚本之后会在下面生成修改脚本的SQL，复制出来运行即可完成修改。如下图所示：

![image-20221112170953061](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/image-20221112170953061.png)





