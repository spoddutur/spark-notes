# Jupiter Spark Setup

Following are the consolidated steps that helped me in successfully installing spark with jupyter:
1. Create venv named `jupyter` using conda (I always maintain separate virtual env's for every different setup):
`~/miniconda3/bin/conda create --name jupyter python=3.4`
2. Activate that environment and start installing our stuff there
```markdown
source activate jupyter
```
3. Installation part:
```markdown
conda install -c conda-forge nb_conda
 $ git clone https://github.com/alexarchambault/jupyter-scala.git
$ cd jupyter-scala
sbt publishLocal
./jupyter-scala --id scala-develop --name "Scala (develop)" --force
=====================================
output: Run jupyter console with this kernel with
  jupyter console --kernel scala-develop

Use this kernel from Jupyter notebook, running
  jupyter notebook
and selecting the "Scala (develop)" kernel.
=====================================
```
4. Let's verify installation. For this, list kernels and we should see the following three: 
```markdown
jupyter kernelspec list
scala-develop    /Users/surthi/Library/Jupyter/kernels/scala-develop
python3          /Users/surthi/miniconda3/envs/jupyter/share/jupyter/kernels/python3
python2          /usr/local/share/jupyter/kernels/python2
```
5. That's it!! Open Jupyter and create spark session. I've used spark 2.1.0 below.

```markdown
jupiter notebook

https://github.com/jupyter-scala/jupyter-scala#spark
import $exclude.`org.slf4j:slf4j-log4j12`, $ivy.`org.slf4j:slf4j-nop:1.7.21` // for cleaner logs
import $profile.`hadoop-2.6`
import $ivy.`org.apache.spark::spark-sql:2.1.0` // adjust spark version - spark >= 2.0
import $ivy.`org.apache.hadoop:hadoop-aws:2.6.4`
import $ivy.`org.jupyter-scala::spark:0.4.2` // for JupyterSparkSession (SparkSession aware of the jupyter-scala kernel)

import org.apache.spark._
import org.apache.spark.sql._
import jupyter.spark.session._

val sparkSession = JupyterSparkSession.builder() // important - call this rather than SparkSession.builder()
  .jupyter() // this method must be called straightaway after builder()
  // .yarn("/etc/hadoop/conf") // optional, for Spark on YARN - argument is the Hadoop conf directory
  // .emr("2.6.4") // on AWS ElasticMapReduce, this adds aws-related to the spark jar list
  // .master("local") // change to "yarn-client" on YARN
  // .config("spark.executor.instances", "10")
  // .config("spark.executor.memory", "3g")
  // .config("spark.hadoop.fs.s3a.access.key", awsCredentials._1)
  // .config("spark.hadoop.fs.s3a.secret.key", awsCredentials._2)
  .appName("notebook")
  .getOrCreate()
  ```
  
###  Happy Coding Spark in Jupyter!!!
