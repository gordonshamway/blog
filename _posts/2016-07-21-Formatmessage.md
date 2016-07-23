---
layout: single
title: Use Formatmessage to make it simple!
comments: True
permalink: tsql_formatmessage
author: True
categories:
- blog
- 2016
- TSQL
---


When you know several programming languages like python for example you know about the power of formatting a string with relative ease with the help of the shortliterals like **%s**. For example
```python
sub1 = 'data engineer'
b = "i am a {0}".format(sub1)
```
Looking at SQL-Server 2012-2014 you have something similar which is called **FORMATMESSAGE()**. Normally it is used to through custom ErrorMessages in the Raise call.
But you can use it also to format messages like in python:

```SQL
DECLARE
	@subject_text nvarchar(255),
	@subject nvarchar(255),
	@ScreenSeverity nvarchar(100),
	@ScreenKey bigint

SET @subject_text = '%s - Error found through screen %s'
SET @subject = FORMATMESSAGE(@subject_text, @ScreenSeverity, CAST(@ScreenKey as nvarchar))	
print @subject
```
Through my journey to this code I found out that one has some limitations with this method:
1. Use only **nvarchar(4000)** as the maximum length for the nvarchar variable in which you put the results of the **FORMATMESSAGE** call. SQL Server will through you an error otherwhise.
2. There is no literalabbreviation for bigint. The least near one is %i but will throw you an error in the case of bigint. I personally workaround this like you see in the example with a conversion to string.

SQL-Server 2016 made some enhancements to this function. You can call it now directly on one string and don´t have to use a seperate variable anymore, as shown:

![](https://www.mssqltips.com/tipimages2/FORMATMESSAGE.jpg)
