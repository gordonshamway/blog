---
layout: single
title: Parsing a Logfile date in SSIS with the help of C#
comments: True
permalink: parsing_logfile_date
author: True
categories:
- blog
- 2016
- SSIS
---

recently I had a problem parsing the following date with SSIS: **"Fri 01Jul16 00:02:59"** as you would write it with python coding would be easy as pie:

```python
res['Timestamp'] = datetime.strptime(res['Timestamp'], '%a %d%b%y %H:%M:%S')
```
Using the coding in C# i thought it would natural to use a basic formating string:

```c#
DateTime date = DateTime.ParseExact(test[2].ToString(), "ddd ddMMMyy HH:mm:ss", System.Globalization.CultureInfo.CurrentCulture);
```

i recieved a lot of errors which all didnt point in the right direction.

Than I tried to use only a substring of the given string to parse which was without the abreviated day in the beginning.

so it turned out to work with the following:

```c#
DateTime date = DateTime.ParseExact(test[2].ToString().Substring(4), "ddMMMyy HH:mm:ss", System.Globalization.CultureInfo.CurrentCulture);
```
well everything went good in the end. 
After I googled a bit I found that it is maybe possible with the customized format with just one d  found in [https://msdn.microsoft.com/de-de/library/8kb3ddd4(v=vs.110).aspx]

I will come back to this in the future.

> **Current Conclusion**:
C# is not good with reocurring dateparts...
