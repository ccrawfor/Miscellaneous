# Log File Analysis Using Spark & Scala

## Description

Log data and it's format can vary across development groups.  These variations can make it difficult to spot and highlight issues with applications.

Complicating this is the lack of tools available for easily analyzing this data and in some corporate environments nearly impossible to acquire.  Often times, one finds themselves painstakingly leveraging some combination of Office Tools (Excel, Access) to help sort through and make sense of the data.

Perl is a nice tool to leverage when parsing large text files.  However I have increasingly found it easier to leverage Apache Spark with Scala.

As an example, during a health check, it was discovered that alert notices were not going out.  A quick check of the log files did not seem to indicate that there were any issues with the adapter or with  communication to the backend and by all appearances it appeared that we were capturing the data along with the values that should trigger an alert.

### Log snippet


```

[2017-10-24 21:19:29,697 - plc.py:18 - run ] Key: M5.VFD.OutFreq & value: 99
[2017-10-24 21:19:29,698 - adapter.py:199 - publishSB ] [{"name": "main_drive_speed", "value": 99}]
[2017-10-24 21:19:29,698 - adapter.py:88 - __on_publish ] MESSAGE SENT: 9 : Using Client: <paho.mqtt.client.Client object at 0x7f5f3c73a390>
[2017-10-24 21:19:29,699 - plc.py:18 - run ] Key: M4.VFD.OutCur & value: 170
[2017-10-24 21:19:29,699 - adapter.py:199 - publishSB ] [{"name": "loader_drive_current", "value": 170}]
[2017-10-24 21:19:29,699 - adapter.py:88 - __on_publish ] MESSAGE SENT: 10 : Using Client: <paho.mqtt.client.Client object at 0x7f5f3c73a390>

```

We know from our log analysis that we print out the message's data right before we receive the acknowledgement that the message has been sent.  

```
[2017-10-24 21:19:29,698 - adapter.py:199 - publishSB ] [{"name": "main_drive_speed", "value": 99}]
[2017-10-24 21:19:29,698 - adapter.py:88 - __on_publish ] MESSAGE SENT: 9 : Using Client: <paho.mqtt.client.Client object at 0x7f5f3c73a390>

```

We need the ability to pair the message's data with it's acknowledgment.  To accomplish this we will use the sliding member on the List class which is defined as:

```
def sliding(size: Int, step: Int): Iterator[List[A]]

Groups elements in fixed size blocks by passing a "sliding window" over them (as opposed to partitioning them, as is done in grouped.)

```

For illustration purposes let's take a look at an example.  As the member description above states we have two choices we can take.

##### 1.  We can use "sliding"
##### 2.  We can use "grouped"  

Again using the sample log snippet below.

```

[2017-10-24 21:19:29,697 - plc.py:18 - run ] Key: M5.VFD.OutFreq & value: 99
[2017-10-24 21:19:29,698 - adapter.py:199 - publishSB ] [{"name": "main_drive_speed", "value": 99}]
[2017-10-24 21:19:29,698 - adapter.py:88 - __on_publish ] MESSAGE SENT: 9 : Using Client: <paho.mqtt.client.Client object at 0x7f5f3c73a390>
[2017-10-24 21:19:29,699 - plc.py:18 - run ] Key: M4.VFD.OutCur & value: 170
[2017-10-24 21:19:29,699 - adapter.py:199 - publishSB ] [{"name": "loader_drive_current", "value": 170}]
[2017-10-24 21:19:29,699 - adapter.py:88 - __on_publish ] MESSAGE SENT: 10 : Using Client: <paho.mqtt.client.Client object at 0x7f5f3c73a390>

```

##### Sliding Window Output:

Notice that we capture the data for **MESSAGE SENT: 9 & 10**

```
main_drive_speed,99,2017-10-24 21:19:29.000
loader_drive_current,170,2017-10-24 21:19:29.000

```
However if we **"grouped"** the list we would end up missing messages that were sent and acknowledged.  Here we are missing the data for **MESSAGE SENT: 9**.

```
loader_drive_current,170,2017-10-24 21:19:29.000

```

To bring this all together we make a dataset from the adapter log.

```
val logFile = sc.textFile("c:/tmp/plc1.log")
val regX2 = raw"(.*?)(\bMESSAGE SENT\b)(.*)".r.unanchored
```

Once our log output is represented as a dataset we can then process it by converting it to a logical grouping of elements and process each set by extracting the data necessary to compare with the data stored on the cloud.

```
def msgProc(cal:List[Any]) {

   var date = ""
   var v = ""
   var sb = new StringBuilder()
   val ack = regX2.findFirstIn(cal(1).toString).getOrElse("") 
   if (ack != "") {
      val pub = (cal(0).toString).replace("{", "").replace("}", "").replace("[","").replace("]","").replace("\"","")
      val v = new Regex("""(value:([^,]*))""", "name", "value")
      val n = new Regex("""(name:([^,]*))""", "name", "value")
      val t = new Regex("""(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})""", "value")
      val nr = n.findFirstMatchIn(pub)
         if (nr.isDefined){
               sb.append(nr.get.group("value")).append(",")    
         }
         val vr = v.findFirstMatchIn(pub)
         if (vr.isDefined){
               sb.append(vr.get.group("value")).append(",")      
         }
         val tr = t.findFirstMatchIn(pub)
         if (tr.isDefined){
               
               val regX = new Regex("""(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})""", "value")
               val date = regX.findFirstIn(tr.get.group("value"))
               if (date.isDefined) {
                sb.append(date.get).append(".000")
               }      
         }
         
          println(sb.toString())
    }
  
}       

```
##### Sliding Window Execution:

```
logFile.collect.toList.sliding(2).foreach(x => msgProc(x)) 
```

##### Grouped Window Execution:

```
logFile.collect.toList.grouped(2).foreach(x => msgProc(x)) 
```

At this point we can follow a similar path for converting data captured in the cloud and subsequently convert our dataset(s) into dataframes and run SQL queries programmatically against them.

```
val sqlDF = spark.sql("SELECT * FROM cloud WHERE tag = 'main_drive_speed'")

+----------------+-----+--------------------+
|             tag|value|                time|
+----------------+-----+--------------------+
| main_drive_s...|  102|2017-10-25 01:17:...|
| main_drive_s...|   99|2017-10-25 01:19:...|
| main_drive_s...|   11|2017-10-25 01:21:...|
| main_drive_s...|   21|2017-10-25 01:23:...|
| main_drive_s...|   37|2017-10-25 01:25:...|
| main_drive_s...|  139|2017-10-25 01:27:...|
| main_drive_s...|  138|2017-10-25 01:28:...|
| main_drive_s...|   99|2017-10-25 01:29:...|
| main_drive_s...|   10|2017-10-25 01:30:...|
| main_drive_s...|   57|2017-10-25 01:32:...|
| main_drive_s...|  136|2017-10-25 01:34:...|
| main_drive_s...|   85|2017-10-25 01:36:...|
| main_drive_s...|  171|2017-10-25 01:38:...|
| main_drive_s...|   24|2017-10-25 01:40:...|
| main_drive_s...|  184|2017-10-25 01:41:...|
| main_drive_s...|  176|2017-10-25 01:42:...|
| main_drive_s...|  164|2017-10-25 01:43:...|
| main_drive_s...|  175|2017-10-25 01:45:...|
| main_drive_s...|   84|2017-10-25 01:47:...|
| main_drive_s...|   39|2017-10-25 01:49:...|
+----------------+-----+--------------------+

``` 
