---
layout: single
title: Check if oozie is running properly with an example!
comments: True
permalink: hadoop_oozie_check
author: True
categories:
- blog
- 2016
- Oozie
---

Hi guys,
If you have oozie installed or it was installed on a instance where you are working with it is always nice to make sure that everything is properly working. Also you have to find a way all around the big data infrastructure that you´re working with.
So in this example which is based on the excellent video which I found on [Youtube](https://www.youtube.com/watch?v=Y1Fvz9tgdA8&list=PLf0swTFhTI8pH1wHiMMTLQCYFSIhrBAqZ&index=2)

## Goal
find out if oozie is working correctly by using the examples deployed with the package. In this blogpost we are going to use the simple java-main example which is a simple map job to find out if everything is fine.

## 1. Do we have the examples?
to check if we have the examples I´m talking about we drop a simple bash command on our favourite terminal:
```bash
find /usr -name "oozie*examples*"
```
you will find the following answer:
![](http://i.imgur.com/XGgrr0p.png)

**/usr/hdp/2.4.0.0-169/oozie/doc/oozie-examples.tar.gz** on your local harddrive.

NOTE: with the find command you can also see examples for spark!

{% capture notice-1 %}
#### Note:
with the find command you can also see examples for spark!
{% endcapture %}
<div class="notice">{{ notice-1 | markdownify }}</div>

## 2. Get the examples to a place which is comfortable to us
We copy the found file to a local directory and unpack it:
```bash
mkdir demo
cd demo
cp /usr/hdp/2.4.0.0-169/oozie/doc/oozie-examples.tar.gz .
tar xvzf oozie-examples.tar.gz
cd examples/apps/java-main
```
the application code itself exists in the lib folder underneath the current one which is named **oozie-examples-4.2.0.2.4.0.0-169.jar**
so now we are in the correct folder which we want to copy to hdfs

## 3. Find the relevant information to configure and run this: 
To make a correct config file we should know the following things in advance:
**- server which is the oozie host**
**- namenode**
**- jobtracker**


#### Find out the **oozie server**
1. click on ambari server website (port 8080) on services
2. click on Oozie on the left hand side
3. finally click on configs

unfortunatly searching for the value of interest doesnt work in this case. 
![](http://i.imgur.com/HV2Gj41.png)

To overcome this issue start your browsers search function and search for the marked string: oozie.base.url and remind yourself on this value. Mine is: http://ambari2:11000/oozie

![](http://i.imgur.com/51SeHoZ.png)


#### How to get the variable content of  the **namenode**:
1. click on your ambari server website (port 8080) on Services
2. Click on HDFS on the left hand side
3. inside of the search field type fs.default to get the adress of the namenode as shown below:
 ![](http://i.imgur.com/KuGzTRm.png)


### How to get the variable content of the **jobtracker**:
Because I use my cluster with yarn my jobtracker is found in the yarn configuration of HDP:
to get the correct information for our second variable do the following clicks:
1. click on your ambari server website (port 8080) on services
2. click on Yarn on the left hand side
3. click on the configs if it isnt already active
4. type in resourcemanager.address in the searchfield
5. find the correct adress as shown below:
![](http://i.imgur.com/rNWdFBs.png)

now we can use the just found adress to replace the given value in **/demo/examples/apps/java-main/job.properties** file that I showed above.

after you plugged in your values the job.properties 
like I did the same with:
- namenode: hdfs://ambari1:8020
- jobtracker: ambari2:8050

file should be looking like this:
![](http://i.imgur.com/G4816BZ.png)

## 4. Copy the files to hdfs
Now we are going to copy the files which are in the examples folder to hdfs so that the cluster can run this properly:

```bash
cd ~
cd demo
hadoop fs -put examples /user/hadoop/examples
```


{% capture notice-2 %}
#### Note:
Keep in mind that /user/**hadoop** -> is my user, you might have a different one.
{% endcapture %}
<div class="notice">{{ notice-2 | markdownify }}</div>


## 5. Run the testjob:

If you wanna run the testjob you have to use the oozie host after the **-oozie** parameter followed by the **local** job.properties file which you configured above.
```bash
oozie job -oozie http://ambari2:11000/oozie -config /home/hadoop/demo/examples/apps/java-main/job.properties -run
```

![](http://i.imgur.com/Oddbpf0.png)


NOTE
if you get the error:
**/usr/hdp/2.4.0.0-169/oozie/bin/oozie.distro: line 59: /usr/bin/java/bin/java: Not a directory**
that means that you probably configured your $JAVA_HOME wrong. Set it to /usr and retry.

{% capture notice-3 %}
#### Note:
If you get the error:
**/usr/hdp/2.4.0.0-169/oozie/bin/oozie.distro: line 59: /usr/bin/java/bin/java: Not a directory**
that means that you probably configured your $JAVA_HOME wrong. Set it to /usr and retry.
{% endcapture %}
<div class="notice">{{ notice-3 | markdownify }}</div>

## 6. check if everything worked out
Once you send out the previous command out to the world you got back a jobid from oozie. with this as a parameter you can check out if everything worked out fine, like so for a command line approach:
```bash
oozie job -oozie http://ambari2:11000/oozie -info JOBID
```

![](http://i.imgur.com/a75R8Fq.png)

or so for a web based approach:

1. Go to your webbrowser and open the url of the jobtracker that you found out in the begining for example (just the port will be 8080): For me it was: **ambari2:8080/cluster** where you can see all the submitted jobs:

![](http://i.imgur.com/q1hgEtH.png)
you can see that it succeeded. 
	If you click on the jobnumber you see in the next screen:
![](http://i.imgur.com/AhXcSL8.png)

 Now click on history:
![](http://i.imgur.com/3xIcDRb.png)
Because this job had only one map task you can click on it and you see the job information e.a. who long it took.
The output of the step is visible if you click on the logs in the following screen:

![](http://i.imgur.com/s1MVSxg.png)

and can see the output which in this case was nothing :-)

![resultset](http://i.imgur.com/h0oJhWU.png)

That was it I hoped everything worked out for you. If you got some tips or questions, just let me know in the comments.
