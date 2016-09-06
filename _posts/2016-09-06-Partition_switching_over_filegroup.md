---
layout: single
title: Partitionswitching over filegroups!
comments: True
permalink: Partition_switching_over_filegroup
author: True
categories:
- blog
- 2016
- TSQL
---


Normally there is a concept of partitioning which uses the **RIGHT** method of partitioning. If there is the need to move partitions from one filegroup to another - since the filegroups should be a moving data period - one needs an efficent way to do this.
Mostly you want to do this with at least physical file operations than you can afford. Even the latter is the hardest to aim for. Olthough filegroups are used often by admins to ease their work with databases and backups.

In this article I want to guide you a way through this topic as I learned it from [Dan Gutzmans Blog Post](http://weblogs.sqlteam.com/dang/archive/2011/04/17/move-a-partition-to-a-different-file-group-efficiently.aspx)

The following example is just to show you the concepts of that topic. I hope that you can take the ideas of this code and use it for your problem at hand.

Think that we have two filegroups as you can see in the picture below. Next imagine that tomorrow is new year and we have to change the data from the **last_years** filegroup into the **archive** filegroup.

![](http://i.imgur.com/NtUeP89.png)

We want to move partition **5711** from the **last_year** filegroup to the right of partition **4715** in the **archive** filegroup



>#### Info
>Since we only show the partition movement for one partition in last_year on purpose because it is the same for every partition this >should be enough for your clarification!



Now the steps to get to our aim should be as follows:

### 1. Step - Preparation:
Move the partition **5711** on a new to be created table **temp_new_to_old** on the filegroup **last_year**. Also we have to create also a table named **temp_old_to_old** on filegroup **archive** to support the move of the partition **4715** 

![](http://i.imgur.com/47kPR1z.png)

```SQL
CREATE TABLE temp_old_to_old (
orderid int not null
, orderdate datetime not null
)
on archive

CREATE TABLE temp_new_to_old (
orderid int not null
, orderdate datetime not null
)
on last_year;
```

You have to take in mind that this table that we just created are only vehicles to move the data between filegroups. When you finished this code you can delete them with no problem.
Very important here is that you have noticed that the temp tables that we created go to their respective filegroup where the related source is coming from (see the picture to check)

```SQL
CREATE UNIQUE CLUSTERED INDEX PK_Orders_old_to_old
on temp_old_to_old(orderid)
ON archive

CREATE UNIQUE CLUSTERED INDEX PK_Orders_new_to_old
on temp_new_to_old(orderid)
ON last_year
```

One prequesit to move the data between partitions is that the indexes and constraints on the different entities should be aligned. For example think that the values of Partition **5711** are only between 10 and 20 then you have to create a constraint where this range is checked!

```SQL
ALTER TABLE temp_new_to_old
ADD CONSTRAINT CK_a CHECK (ORDERID >= 10 AND Orderid < 20)
```

### 2. Step - Move data out :
After both tables have been created the content of the Partitioned_Table can be transferred to the both tables.

```SQL
ALTER TABLE Partitioned_Table
SWITCH Partition $Partition.pf_Partitionfunction(some_value inside of partition 4715)
TO temp_old_to_old;

ALTER TABLE Partitioned_Table
SWITCH PARTITION $PARTITION.pf_Partitionfunction(some value of partition 5711)
TO temp_new_to_old
```

Both partitions of **Partitioned_Table** are now empty. Now we can use this situation and change the partition function and so on because those operations are very very fast on empty partitions. This is also the reason why we moved partition **4715** also, which otherwhise were not necessary.

### 3. Step - Change the partitions:
We will change the partition function so that partition **5711** will be merged. What that means is when operated by a right oriented partitionfunction we will delete the partition boundary **5711** and all related data will go to the next left partition which is then in charge for the data (theoretically **4715**)

```SQL
ALTER PARTITION FUNCTION [pf_Partitionfunction]()
MERGE RANGE( Partitionboundary 5711 );
```

>#### Warning
>We havent moved data yet, we just touched structers so far in step 3

Next we will make sure that the **archive** filegroup is used, so we just take action and do this:

```SQL
ALTER PARTITION SCHEME [ps_Partitionschema]
NEXT USED archive;
```

This goes in conjunction with Step 4.

### 4. Step - Create a new Partition boundary:
What we have to do now is to split the partition function again and insert the boundary that we have deleted with the merge statement. The newly created partition will be then created on the **archive** filegroup.

```SQL
ALTER PARTITION FUNCTION [pf_Partitionfunction]()
SPLIT RANGE( Partitionboundary 5711 );
```

When you look now on the picture you can see what we have done just now - moved the borders and increased the size of the **archive** filegroup and extended it with another partition.

![](http://i.imgur.com/Z5kaua5.png)

You may say: "Yeah" now we have to just move the data back to the **Partitioned_table** again to have our job done. But there is one little trick to make this work and don´t get an error: "Msg 4939, Level 16, State 1, Line 666
ALTER TABLE SWITCH statement failed. index 'AAA' is in filegroup 'BLABLABLA' and partition X of index 'BBB' is in filegroup 'BLABLABLA'." which I will tell you just now!

### 5. Step - do the trick:
Since it is not so easy to move partitions or tables between filegroups without masses of physical IOs. We just move an index and data will follow! 
We created this index **PK_Orders_new_to_old** in step one on our temporary table. We are going to recreate it on the filegroup **archive** and we are done. I will show you how the magic happens:

```SQL
CREATE UNIQUE CLUSTERED INDEX PK_Orders_new_to_old
on temp_new_to_old(orderid)
WITH (DROP_EXISTING = ON) --> here happens the magic
ON archive --> here happens the magic
```

### 6. Step - Move back:
After we changed the location of the index just move the data back to the **Partitioned_Table** and are done. I can´t await it, so let´s do it!

![](http://i.imgur.com/henwSYs.png)

```SQL
ALTER TABLE temp_old_to_old
SWITCH TO Partitioned_Table PARTITION $PARTITION.pf_Partitionfunction( Partitionboundary 4715 )

ALTER TABLE temp_new_to_old
SWITCH TO Partitioned_Table PARTITION $PARTITION.pf_Partitionfunction( Partitionboundary 5711 )
```

NOW YOUR DONE! Oh hold on stop it,

maybe you want to add a new partition for the newest data, so this is totally optional. But you could do:

```SQL
ALTER PARTITION SCHEME Partitionschema
NEXT USED last_year(newest id);
ALTER PARTITION FUNCTION Partitionfunction()
SPLIT RANGE (last_value);
```
