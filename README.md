# TP4 Lab2 - MapReduce2/Yarn - Java Jobs

## 1.1. Installing OpenJDK

Using Linux Ubuntu, we can install OpenJDK 8 using ```sudo apt-get install openjdk-8-jdk```. Once installed, we can switch the currently used version to Java 8 using the ```update-alternatives --config java``` command:

```
~$ sudo update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-14-openjdk-amd64/bin/java      1411      auto mode
  1            /usr/lib/jvm/java-14-openjdk-amd64/bin/java      1411      manual mode
  2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

Press <enter> to keep the current choice[*], or type selection number: 2

update-alternatives: using /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
to provide /usr/bin/java (java) in manual mode
```

## 1.2. Installing Git

We can install git with the command ```sudo apt-get install git```. We will install hub as well to simplify the process of creating the repository. To install hub, we can use Homebrew (https://docs.brew.sh/Homebrew-on-Linux) and execute ```brew install hub```.

## 1.3. Installing a Java IDE

We will code using Eclipse (installed via ```sudo snap install --classic eclipse```).

## 1.4. Cloning the project

To use hub, one needs to create a SSH key associated with their GitHub account. Once the key is added, we can clone the original repository (```git clone https://github.com/makayel/hadoop-examples-mapreduce```) and create our own repository derived from it using the ```hub create``` command, followed by ```git push -u origin main``` (setting the origin of the newly created repository to the "main" branch of the project, the renamed master branch).

## 1.5. Importing the project

We import the project using Eclipse's Maven project importer : ```File > Import... > Maven > Existing Maven Projects```, ```Browse > Open```, select the ```pom.xml``` and press ```Finish```.

To generate the JAR, we can launch the maven install goal in the start directory, running the command ```mvn install``` (can also be executed using the M2Eclipse plugin). To install maven, use ```sudo apt install maven```.

## 1.6. Send the JAR to the edge node

To simplify the process of importing the target JAR to our edge node, we will directly clone the repository of our project in our local cloud storage (```https://github.com/ivan-stepanian/hadoop-examples-mapreduce```) and generate the new target files with maven directly on the edge (we could also generate it before, and use ```git pull``` to retrieve the target folder, were it to be included in the repository). To modify the source java files, we will use Eclipse on our machine, `commit` and `push` the changes, and `pull` the on the edge node before regenerating the jar. Using Linux, we will directly connect to the edge node via `ssh`.

## 1.7. The repository

You can find our repository at https://github.com/ivan-stepanian/hadoop-examples-mapreduce and visualize all the commits.

## 1.8. The Project

We first download the dataset using wget on the edge node, and put it on the HDFS:
```
-sh-4.2$ wget https://raw.githubusercontent.com/makayel/hadoop-examples-mapreduce/main/src/test/resources/data/trees.csv
--2020-11-10 17:21:36-- https://raw.githubusercontent.com/makayel/hadoop-examples-mapreduce/main/src/test/resources/data/trees.csv
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.120.133
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.120.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16680 (16K) [text/plain]
Saving to: ‘trees.csv’

100%[======================================>] 16 680      --.-K/s   in 0,001s  

2020-11-17:22:01 (15,6 MB/s) - ‘trees.csv’ saved [16680/16680]

-sh-4.2$ hdfs dfs -put trees.csv

-sh-4.2$ hdfs dfs -ls
Found 18 items
...
-rw-r--r--   3 istepanian hdfs     121271 2020-11-10 17:22 trees.csv
```

Using the default job already implemented in the target JAR (wordcount), we can test it on our dataset (we will create an alias, `launch_jar`, to avoid launching long commands):

```
-sh-4.2$ alias launch_job="yarn jar ~/hadoop-examples-mapreduce/target/hadoop-examples-mapreduce-1.0-SNAPSHOT-jar-with-dependencies.jar"

-sh-4.2$ launch_job wordcount trees.csv count
...
20/11/10 17:27:55 INFO mapreduce.Job: Running job: job_1603290159664_3340
...
20/11/10 17:28:06 INFO mapreduce.Job:  map 0% reduce 0%
20/11/10 17:28:15 INFO mapreduce.Job:  map 100% reduce 0%
20/11/10 17:28:20 INFO mapreduce.Job:  map 100% reduce 100%
...
	File Input Format Counters
		Bytes Read=121271
	File Output Format Counters
		Bytes Written=68405

-sh-4.2$ hdfs dfs -cat count/part-r-00000
...
·	2
à	13
écus;;10;Parc	1
écus;;31;Jardin	1
écus;;46;Parc	1
écus;;64;Bois	1
écus;;84;Bois	1
île	1
...
```

Everything works correctly, the job completed successfully.

## 1.8.1. DistrictsWithTrees

For this MapReduce job, we create a simple job based on the files from the previous job `wordcount`, creating a job class `DistinctDistricts`, a mapper class `TreesMapper` and a reducer class `TreesReducer`

We then add our class to the `AppDriver` so the program can interpret our new command `distinctDistricts`:

```
programDriver.addClass("distinctDistricts", DistinctDistricts.class,
        "A map/reduce program that returns the distinct districts with trees in a predefined CSV formatting.");
```

We can also add a custom command description in case of a wrong typing:

```
if (otherArgs.length < 2) {
    System.err.println("Usage: distinctDistricts <in> [<in>...] <out>");
    System.exit(2);
}
```

Then, we set our mapper and reducer classes for the job in `DistinctDistricts`:

```
Job job = Job.getInstance(conf, "distinctDistricts");
job.setJarByClass(DistinctDistricts.class);
job.setMapperClass(TreesMapper.class);
job.setCombinerClass(TreesReducer.class);
job.setReducerClass(TreesReducer.class);
```

Because the only information we will require is the district number, all we need is a `Text` key (that will contain the name of the district) with a `null` (`NullWritable`) value. By default, because of the way the MapReduce programming model works, all the keys will be made distinct, and we will get an `Iterable` of `NullWritable` instances aggregated. We can then just return the keys, as they will all be the distinct districts

```
job.setOutputKeyClass(Text.class);
job.setOutputValueClass(NullWritable.class);
```
