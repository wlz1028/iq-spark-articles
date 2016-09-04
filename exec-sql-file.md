# [Tutorial] How to execute sql file using Spark/Scala

In this tutorial, I will show you how to execute sql/HiveQL files from Spark/Scala. HDFS is not necessary in this design.

## Prerequisites / Audiences 
* You have experience in Hive, Oozie, and Spark
* You have some knowledge in Spark/Scala
* source code: https://github.com/wlz1028/iq-spark
* My environment:
	* Spark 1.6.0 HDP version (please refer to pom.xml)
	* No Hadoop or HDP sandbox
	* Intellij
	* Maven
	* Mac OS

## Use cases
* You are migrating from Hive to Spark
* You want to only maintain sql files without modifying Scala code

## Sample data and sql
Let's say we have the following bank transaction data in csv format.

	trans_ctgy,cust_id,trans_amt,trans_date
	restaurant,1,50,20160701
	hotel,1,200,20160702
	restaurant,1,30,20160710
	education,1,50,20160711
	hotel,1,200,20160715
	music,1,300,20160802
	music,1,300,20160803
	restaurant,2,15,20160710
	restaurant,2,35,20160715
	  
`q1.sql` select transactions from July 2016 <a name="q1"></a>

```sql
SELECT *
FROM source_transaction_table
where substring(trans_date,1,6)=201607
```

##sql files location

Sql files are placed into `src/main/resources/sql/` which is packaged into jar file by Maven/SBT. In this way, your sql files are always available to your Spark/Scala code. It's also easier to deploy since all you need is to upload modified jar file to HDFS/s3. The following code snippet demonstrate how to read a file from jar file using relative path.

Source code is available on my Github: <https://github.com/wlz1028/iq-spark>.

```scala
package ca.infoq.spark.common.io

object IoUtils {
  //Use relative path (e.g. sql/q1.sql)
  val getFileStrFromResource = (path: String) => scala.io.Source.fromInputStream(this.getClass.getClassLoader.getResourceAsStream(path)).mkString
}
```

Alternatively, you could also upload your sql files to HDFS instead of placing into jar file. But you have to be careful in deployment since Spark/Scala App and sql files are decoupled.

## Variable substitution (e.g. hive -hivevar)
Variable substitution is a builtin feature in most sql engines (e.g. Apache Hive  <https://cwiki.apache.org/confluence/display/Hive/LanguageManual+VariableSubstitution>, MySql<http://dev.mysql.com/doc/refman/5.7/en/user-variables.html>). However, this is a missing feature in SparkSQL. In the next post, I will show you how to implement variable substation in Scala with a few lines of code.


