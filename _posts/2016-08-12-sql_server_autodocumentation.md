---
layout: single
title: Auto document your work with SQL-Server!
comments: True
permalink: sql_server_autodocumentation
author: True
categories:
- blog
- 2016
- TSQL
---


### Problem:
Have you ever wrote tens or hundreds of stored procedures where you use a lot of functions, Tables, other Databases and stuff like this?
Probably the answer is yes!
Now what are you going to do if your boss or customer wants an overview over your work in a graphical form?
Well you could go for it and spend hours search through all the objects and their relations inside and outside of the db.

### Solution:
A faster way for doing this is use the dynamic views of syql_server to auto document your work.
With a little bit of work I will show you how to build a easy to implement and change system, that automatically document your work. 

### Advantages:
1. You can present your work to anybody in no time
2. Your collegues or yourself will no what you did when time went by and you have to change something
3. Dependencies as well as hierarchies are easy to find and to take care on
4. Optional errors are also more visible to you for later checking

### Requirements:
1. SQL Server 
2. (its dynamic views: sys.dm_sql_referenced_entities, sys.objects, sys.schemas)
3. [yEd graph editor](http://www.yworks.com/)
4. SSIS or any other ETL Tool a.e. Pentaho Data Integration to automate the procurement process

### Basic description of our procedure:
1. We will use yEd as a visualization tool for our data. This tool is capable of importing Excel and CSV data in an automated manner where you can set up different styles for the columns.
2. Create an Excel file with two kinds of information:
	3. All objects and their regarding descriptive informations like name, schema, database, type of object and other information that you want to visualize for example color
	4. The relations that exist between the objects like which stored procedure is writing in which table or takes data from which view or is manipulating stuff with a user defined function
3. To fullfill the needs described above we will use the **sys.objects**, **sys.schemas** and **sys.dm_sql_referenced_entities** objects of SQL-Server. Also we are going to create two working tables to save our findings and use them as target tables for our export process to Excel respective yEd.

### Step 1 - Create working tables

```SQL

/*Check if the tables that we are about to create are not existant*/
IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID('dbo.edge') AND type in (N'U'))
	DROP TABLE dbo.edge;
IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID('dbo.node') AND type in (N'U'))
	DROP TABLE dbo.node;

/* All our relations will go into this EDGE table */
CREATE table EDGE([Source]  varchar(50), [Target] varchar(50), [Weight] int, [Color] varchar(10), [in_out] int, [Schema] varchar(50))

/* All informations regarding the important objects that you want to descripe has to go  into this NODE table */
CREATE TABLE [dbo].[NODE](
	[Database] [nvarchar](128) NULL,
	[Schema] [sysname] NULL,
	[Objectname] [sysname] NOT NULL,
	[Two_part_name] [nvarchar](257) NOT NULL,
	[Three_part_name] [nvarchar](386) NOT NULL,
	[Objecttype] [char](2) NOT NULL,
	[Description] [nvarchar](60) NULL,
	[Color] [nvarchar](50) NULL
)
```


### 2. Step - Load edges

```SQL
DECLARE @ProcedureName varchar(50)
DECLARE @ProcedureServer varchar(50)
DECLARE @ProcedureSchema varchar(50)
DECLARE @ProcedureDB varchar(50)
DECLARE @Color varchar(50)

DECLARE MY_CURSOR CURSOR 
  LOCAL STATIC READ_ONLY FORWARD_ONLY
FOR 
select 'YOURSERVER', SPECIFIC_CATALOG, SPECIFIC_SCHEMA, SPECIFIC_SCHEMA+'.'+SPECIFIC_NAME from information_schema.routines
where routine_type='PROCEDURE' and specific_schema not in (/*'dbo',*/'sys')

OPEN MY_CURSOR
FETCH NEXT FROM MY_CURSOR INTO @ProcedureServer, @ProcedureDB, @ProcedureSchema, @ProcedureName
WHILE @@FETCH_STATUS = 0
BEGIN 
	BEGIN TRY
		SET @Color = '#'+substring(master.dbo.fn_varbintohexstr((CAST(@ProcedureSchema AS VARBINARY))), 4, 6)
		INSERT INTO dbo.EDGE
		SELECT LOWER(referenced_schema_name+'.'+referenced_entity_name) as [Source], LOWER(@ProcedureName) as [Target], count(referenced_minor_name) as [Weight], @Color as [Color], 1 as in_out, LOWER(@ProcedureSchema) as [Schema]
			FROM sys.dm_sql_referenced_entities (@ProcedureName, 'OBJECT') 
			where is_selected = 1 OR is_select_all = 1
			group by referenced_schema_name, referenced_entity_name
		UNION
		SELECT LOWER(@ProcedureName) as [Source], LOWER(referenced_schema_name+'.'+referenced_entity_name) as [Target], count(referenced_minor_name) as [Weight], @Color as [Color], 0 as in_out, LOWER(@ProcedureSchema) as [Schema]
			FROM sys.dm_sql_referenced_entities (@ProcedureName, 'OBJECT')
			where is_updated = 1
			group by referenced_schema_name, referenced_entity_name
		UNION
		
		/*these are procedures where no selects and updates are executed*/
		SELECT LOWER(@ProcedureName) as [Source], LOWER(referenced_schema_name+'.'+referenced_entity_name) as [Target], count(referenced_minor_name) as [Weight], @Color as [Color], 2 as in_out, LOWER(@ProcedureSchema) as [Schema]
			FROM sys.dm_sql_referenced_entities (@ProcedureName, 'OBJECT')
			where is_selected = 0 and is_select_all = 0 and is_updated = 0
			group by referenced_schema_name, referenced_entity_name
	END TRY
	BEGIN CATCH
		Insert into dbo.errormessages_reference 
		SELECT
        ERROR_NUMBER() AS ErrorNumber,
        ERROR_SEVERITY() AS ErrorSeverity,
        ERROR_STATE() as ErrorState,
        ERROR_PROCEDURE() as ErrorProcedure,
        ERROR_LINE() as ErrorLine,
        ERROR_MESSAGE() as ErrorMessage

	END CATCH
    FETCH NEXT FROM MY_CURSOR INTO @ProcedureServer, @ProcedureDB, @ProcedureSchema, @ProcedureName
END
CLOSE MY_CURSOR
DEALLOCATE MY_CURSOR
```

### 3. Step - Load nodes

```SQL
insert into NODE
select 
DB_NAME() as [Database]
,LOWER(b.name) as [Schema]
,LOWER(a.name) as Objectname
,LOWER(CONCAT(b.name,'.', a.name)) as Two_part_name
,LOWER(CONCAT(DB_NAME(),'.',b.name,'.', a.name)) as Three_part_name
,type as Objecttype
,type_desc as [Description] 
/* just to get different colors per name*/
,'#'+substring(master.dbo.fn_varbintohexstr((CAST(b.name AS VARBINARY))), 4, 6) as Color
from 
sys.objects as a 
left join sys.schemas as b
on 
a.[schema_id] = b.[schema_id]
 where 
type not in ('PK','F', 'S', 'D', 'SN', 'UQ', 'IT', 'SQ') 

```

### 4. Step - Export everything to Excel
When you are at this point you can extract the content of the tables node and edge to one Excel file in the regarding sheets with the respective names edge and node.

### 5. Step - Set up yEd 
Once you opened yEd click on **Edit** -> **Properties Mapper... **
Create a new configration for the nodes:
In this place you set up the columns and what you want yEd to do with it. 
For example: 

- I want it to use the column Description (column G in my Excel file)
   to map it to the Template Attribute which means which type of symbol
   is used. In the lower box of the screenshot you can see that I maped
   every SQL-Server Object to a different symbol that will be used when
   the program is drawing the visualization. 
- Also I want that the column Two_part_name will be used as a text on the symbol. 
- Finally  the color is used to colorize the symbol based on the colorvalue in the
   given column.

![set up nodes](http://i.imgur.com/4bxJSoZ.png)

After this create a new configuration for the edges. What you do here is basically set the line type of the column in_out to distinct values in this column for example for value 1 you set a dotted line.

![set up edges](http://i.imgur.com/FfKSFX5.png)

### 6. Step - Read Excel File into yEd
Once you opened the yEd programm you can drag and drop the Excel file into yEd. After you did this the following window will pop up:

![configure edges and nodes](http://i.imgur.com/DVacpgl.png)

In this window you find configurations on the left side and the on the right side you see the contents of the Excel file. 
for the Data Range you select the whole area of data that is relevant for the program from A to F in my case.
After this you have to select Source and Target columns which are column A and B

> **Hint**
> Be careful the name of the sheet will sometimes not change and you will have to do it by hand.

After you set this up do the same for the nodes as you can see on the screenshot.

Finally change the tab to Presentation and you will find the following options:

![configure presentation](http://i.imgur.com/MCsiD1T.png)

Just make sure to check use the configurations for nodes and edges and set a layout that you want.
When setting up the configurations that you are about to use, you are basically using the once we set up above in step 5.


### 7. Step - Automate most of it

If you want a straight forward automated process for it you can use the ETL Tool of your choice to do the heavy work for you. Here I used SSIS for the given task and set up a projekt where I execute the SQL statements that I showed above in the named steps. 
In the *Clear Excel File* step I copy a template Excel file over the target file to have one blank new Excel file to write our outputs to (because Microsoft is not capable of overwriting its Excel Files in SSIS).
In the put *edges / nodes* into excel I just put the query results into the respective sheets.

Lastly in the final step I just delete my tables to clean up the database, because I dont need  this anymore. This is a optional step so you don´t need to do this.

![ssis steps](http://i.imgur.com/EYtmANj.png)

Once you do all this you can get your auto documented work as you can see here:

![complete graph](http://i.imgur.com/whc7AOg.png)

I just zoomed out of this that you can see that the program yEd is capable of a lot of stuff.
When you need just a zoomed in version you have two options: 
1. Zoom in
2. Use the Neighborhood view where you can see the direct source and target of a symbol when you click on it.
 ![neighborhood menu](http://i.imgur.com/ZSsMQ2L.png)
### Conclusion
You just have to execute the ETL process and the rest is you step 6 where you import the new data and set up the 6 fields with the columns you need. Everything else will be done for you automatically.
I hope this helps you in creating a nice procedure to autodocument your work.

> Actually the dynamic view: **sys.dm_sql_referenced_entities** will not include table references when you use: 
```SQL
SELECT
*
INTO B --> this table will be ignored
FROM A
```

If you have any questions or concerns, don´t hesitate to leave me a comment.
