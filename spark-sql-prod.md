# [Tutorial] How to migrate legacy Hive to Spark using a simple framework (without using Hive on Spark)

Apache Hive has been widely used in many corporations in ETL and some data analytics because of its easy-to-use SQL interface and BigData processing capability. However, as Apache Spark community is thriving, you might consider switching from Hive to SparkSQL. In this article, I will demostrate a small Spark/Scala framework which allows you to execute the **exact same** Hive queries/scripts you already developed using SparkSQL. Furthermore, variable substitution in Spark will also be discussed (similar to -hivevar or -hiveconf feature in Hive).

## Prerequisites / Audiences 
* You have experience in Hive, Oozie, and Spark
* You have some knowledge in Spark/Scala
* iq-spark framework source code: https://github.com/wlz1028/iq-spark
* My environment:
	* Spark 1.6.0 HDP version (please refer to pom.xml)
	* No Hadoop or HDP sandbox
	* Intellij
	* Maven
	* Mac OS


## Why not Hive on Spark
The short answer is that you are **not getting the most out of Spark** (sparkSQL to be more precise). SparkSQL has its own powerful query optimizer/processor called Catalyst(<https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html>). On the other hand, Hive on spark uses Spark core (not sparkSQL) as MapReduce or Tez alternative (<https://issues.apache.org/jira/browse/HIVE-7292>). Here is a **Hive vs SparkSQL vs Hive-on-spark** performance benchmark: <https://hivevssparksql.wordpress.com/>

## Background / Pattern
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
	
The following three simple Hive SQL queries will be execute one after another in order to get the final result. This is a very common pattern in both ETL and data engineering/analytics. 
> For demo purpose, I write three quires instead of one.  
> I also ignored Hive `INSERT INTO TABLE` statements
  
`q1.sql`: Filter source data by given `target_YYYYMM`<a name="q1"></a>

```sql
SELECT *
FROM source_transaction_table
where substring(trans_date,1,6)=${target_YYYYMM}
```

`q2.sql`: Given `cust_id`, get transaction amount for each `trans_ctgy`<a name="q2"></a>

```sql
SELECT trans_ctgy, cust_id, sum(trans_amt) as ctgy_amt
FROM q1_result_table
WHERE cust_id=${id}
group by cust_id, trans_ctgy
```

`q3.sql`: Get `trans_ctgy`<a name="q3"></a>

```sql
SELECT max(ctgy_amt) as amt
FROM q2_result_table
```

Since we want our queries to be generic(e.g. date and ID independent), `${target_YYYYMM}` and `${id}` variables are passed at runtime. For instance, you could setup an Oozie coordinator and workflow to run the same set of queries at given frequency automatically without modifying anything. In Hive, variable substitution is a builtin feature, e.g. -hiveconf and -hivevar (e.g. https://cwiki.apache.org/confluence/display/Hive/LanguageManual+VariableSubstitution). Unfortunately, SparkSQL doesn't support variable substitution, so I will show you how to parse `$variable` in Spark/Scala.

## Spark framework design
Our goal is to execute those three query files using Spark/Scala. Furthermore, sql `$variables` will be passed as Spark App arguments which mimics variable substitution feature in Hive (e.g. `bash> hive -hivevar target_YYYYMM=201605 -hivevar id=1 -f q1.sql`)

Before jumping into step by step implementation, here is a sample App demonstrates how to execute the three sql files with variable substitution using our simple Scala framework at high level. (`Line 31 -40` execute the [q1.sql](#q1),[q2.sql](#q2), and [q2.sql](#q2).)

```
bash> ./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  target_YYYYMM=201607 ID=1
```
<a name="sampleapp"/>
 
```scala
package ca.infoq.spark.app.hivesql

import ca.infoq.spark.common.config.{IqApp, IqAppArgsParser}
import ca.infoq.spark.common.sql.IqSqlUtils.runSql
import ca.infoq.spark.common.io.IoUtils.getFileStrFromResource
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.SQLContext
import org.apache.log4j.Logger
import org.apache.log4j.Level

object IqSqlApp {
  def main(args: Array[String]) {
  	//Parse Spark args to Map 
    val confMap = IqAppArgsParser.getArgsMap(args)
    
    val conf = new SparkConf().setMaster("local[*]").setAppName("IqSqlApp")
    implicit val sc = new SparkContext(conf)
    implicit val sqlContext = new SQLContext(sc)

    val transFile = "src/main/resources/data/fakeTransaction.csv"

    //Load sample csv data
    sqlContext.read.format("com.databricks.spark.csv")
      .option("header", "true")
      .load(transFile)
      .registerTempTable("source_transaction_table")

    //rename method to a shorter name
    def getSqlStr = getFileStrFromResource

	//run sql with pre-defined vars and then registerTempTable and cacheTable
    runSql(getSqlStr("sql/q1.sql"), confMap, "q1_result_table", true)
    sqlContext.table("q1_result_table").show()

    //run another sql from previous result
    runSql(getSqlStr("sql/q2.sql"), confMap, "q2_result_table")
    sqlContext.table("q2_result_table").show()

    //Another demo on sql variableSubstitution which similar to hiveconf/hivevar
    runSql(getSqlStr("sql/q3.sql"), confMap, "q3_result_table")
    sqlContext.table("q3_result_table").show()
    
    //How to uncacheTable
    sqlContext.uncacheTable("q1_result_table")
  }
}

//Sample output result form console
+----------+-------+---------+----------+
|trans_ctgy|cust_id|trans_amt|trans_date|
+----------+-------+---------+----------+
|restaurant|      1|       50|  20160701|
|     hotel|      1|      200|  20160702|
|restaurant|      1|       30|  20160710|
| education|      1|       50|  20160711|
|     hotel|      1|      200|  20160715|
|restaurant|      2|       15|  20160710|
|restaurant|      2|       35|  20160715|
+----------+-------+---------+----------+

+----------+-------+--------+
|trans_ctgy|cust_id|ctgy_amt|
+----------+-------+--------+
| education|      1|    50.0|
|     hotel|      1|   400.0|
|restaurant|      1|    80.0|
+----------+-------+--------+

+-----+
|  amt|
+-----+
|400.0|
+-----+
```

### sql files location
In this design, sql files are placed to `src/main/resources/sql/` which will be packaged into jar file by Maven/SBT. In this way, your sql files are always available to your Spark/Scala code. It's also easier to deploy since all you need is your jar file. 

Alternatively, you could also upload your sql files to HDFS. But you have to be careful in deployment since Spark/Scala App and sql files are decoupled. 

### executing sql file
There are two execution interfaces in SparkSQL. The first one is DataFrame interface (e.g. `df.select($"columnName")`) where the second interface is through `sqlContext.sql(sqlText:String): DataFrame`. The later interface fits our design better which we can read sql file from `src/main/resources/sql/` as `String` and then pass to `sqlContext.sql(sqlText:String)`.  

First of all, I will show you how to read file from `src/main/resource/` as string. Source code can be found in my Github: <https://github.com/wlz1028/iq-spark>.
> The following code is for demo purpose.  
> You are not able to execute as it.  
> Please refer to my iq-spark project on Github ( I will post a video if time permit)

```scala
package ca.infoq.spark.common.io

object IoUtils {
  //Use relative path (e.g. sql/q1.sql)
  val getFileStrFromResource = (path: String) => scala.io.Source.fromInputStream(this.getClass.getClassLoader.getResourceAsStream(path)).mkString
}
```
Now you can execute [q1.sql](#q1) like this:

```scala
	scala> val df = sqlContext.sql(getFileStrFromResource("sql/q1.sql")
	scala> df.show()
	//Sample output
	+----------+-------+---------+----------+
	|trans_ctgy|cust_id|trans_amt|trans_date|
	+----------+-------+---------+----------+
	|restaurant|      1|       50|  20160701|
	|     hotel|      1|      200|  20160702|
	|restaurant|      1|       30|  20160710|
	| education|      1|       50|  20160711|
	...
	
```

However, before executing [q1.sql](#q1), you will have to setup `source_transaction_table` as Spark tempTable. Here is how to read csv file and registerTmpTable: 

```scala
	 //Sample data file is provided in iq-spark project
    scala> val transFile = "src/main/resources/data/fakeTransaction.csv"

    //Load sample data
    scala> sqlContext.read.format("com.databricks.spark.csv")
      .option("header", "true")
      .load(transFile)
      .registerTempTable("source_transaction_table")
      
	//Now you can execute q1.sql
	val df = sqlContext.sql(getFileStrFromResource("sql/q1.sql")
```
### variable substitution
If you execute [q1.sql](#q1) from Hive, you would have to setup `$var` (e.g. `shell> hive -hivevar target_YYYYMM=201607 -f q1.sql`). Similarly, you can pass variable keyValue pairs as Spark app argument for instance:

```
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  target_YYYYMM=201607 id=1
```

In the [sample app main](#sampleapp), `ca.infoq.spark.common.config.IqAppArgsParser.getArgsMap` parses `args: Array[String]` into `Map[String, String]` which will be used for sql variable substitution parsing later.

```scala
import ca.infoq.spark.common.config.IqAppArgsParser

object IqSqlApp {
  def main(args: Array[String]) {
    val confMap = IqAppArgsParser.getArgsMap(args)
    ...
```

`IqAppArgsParser` implementes regex to parse CLI arguments into `Map[String, String]`. In addition, you might want to update regex if your have a more complex app which requires other arguments. Alternative, you an also use some configuration libraries, such as typesafe <https://github.com/typesafehub/config>.

```scala
package ca.infoq.spark.common.config

object IqAppArgsParser {
  val regex = new scala.util.matching.Regex("""(\S+)=\b(\S+)\b""")

  def getArgsMap(args: Array[String]): Map[String, String] = {
    val argStr = args.mkString(" ")
    getArgsMap(argStr)
  }

  def getArgsMap(argStr: String): Map[String, String] = {
    (for (regex(k, v) <- regex findAllIn argStr) yield (k, v)).toMap
  }
}
```
We already discussed how to load sample csv data. Now we can execute [q1.sql]($q1) with parsed arguments. 

```scala
	import ca.infoq.spark.common.sql.IqSqlUtils.runSql
	
	//As discussed, this method reads file from src/main/resources/ as String
	def getSqlStr = ca.infoq.spark.common.io.IoUtils.getFileStrFromResource
	 
    //run sql with pre-defined vars and then registerTempTable
    runSql(getSqlStr("sql/q1.sql"), confMap, "q1_result_table")
    sqlContext.table("q1_result_table").show()

    //run another sql from previous result
    runSql(getSqlStr("sql/q2.sql"), confMap, "q2_result_table")
    sqlContext.table("q2_result_table").show()

    //Another demo on sql variableSubstitution which similar to hiveconf/hivevar
    runSql(getSqlStr("sql/q3.sql"), confMap, "q3_result_table")
    sqlContext.table("q3_result_table").show()
```

`ca.infoq.spark.common.sql.IqSqlUtils.runSql` does three things:

* Substitute `$variables` using ca.infoq.spark.common.sql.resolveDollarVar
* Executed sql using `sqlContext.sql(sqlString): DataFrame`
* Register result as Spark temp table using `registerTempTable(tableName: String)`, therefore can be queried by later sql files
* (Optional) If your temp table will be re-used in multiple sql queries/file, you might want to consider to persist tempTable in memory for better performance. Please also make sure to `sqlContext.uncacheTable` immediate after the tempTable is no longer used.

```scala
  def runSql(sqlStr: String, vars: Map[String, String], outTableName: String, persistFlg: Boolean = false)(implicit sqlContext: SQLContext): Unit = {
    sqlContext.sql(resolveDollarVar(sqlStr, vars)).registerTempTable(outTableName)
    if (persistFlg) sqlContext.cacheTable(outTableName)
  }
```

`ca.infoq.spark.common.sql.IqSqlUtils.resolveDollarVar ` implements regex to parse `$var` from sqlStr by passing `vars: Map[String, String]`(which `vars` was parsed from CLI args).

```scala
  def resolveDollarVar(sqlStr: String, vars: Map[String, String]): String = {
    val varPattern = new scala.util.matching.Regex("""(\$\{(\S+?)\})""", "fullVar", "value")

    varPattern replaceAllIn (sqlStr, m => {
      try {
        vars(m.group("value"))
      } catch {
        case e: NoSuchElementException => throw new NoSuchElementException("Error: " + m.group("fullVar") + " cannot be resolved")
      }
    })
  }
```





