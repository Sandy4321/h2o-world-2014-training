# Real-time Predictions With H2O on Storm

This tutorial shows how to create a [Storm](https://storm.apache.org/) topology can be used to make real-time predictions with [H2O](http://h2o.ai).


## Where to find the latest version of this tutorial

* <https://github.com/0xdata/h2o-training/tree/master/tutorials/streaming/storm/storm.md>


## 1.  What this tutorial covers

<br/>

In this tutorial, we explore a combined modeling and streaming workflow as seen in the picture below:

![](h2o_storm.png)

We produce a GBM model by running H2O and emitting a Java POJO used for scoring.  The POJO is very lightweight and does not depend on any other libraries, not even H2O.  As such, the POJO is perfect for embedding into third-party environments, like a Storm bolt.

This tutorial walks you through the following sequence:

*  Installing the required software
*  A brief discussion of the data
*  Using R to build a gbm model in H2O
*  Exporting the gbm model as a Java POJO
*  Copying the generated POJO files into a Storm bolt build environment
*  Building Storm and the bolt for the model
*  Running a Storm topology with your model deployed
*  Watching predictions in real-time

(Note that R is not strictly required, but is used for convenience by this tutorial.)


## 2.  Installing the required software

### 2.1.  Clone the required repositories from Github

`$ git clone https://github.com/apache/storm.git`  
`$ git clone https://github.com/0xdata/h2o-training.git`  

* *NOTE*: Building storm (c.f. [Section 5](#BuildStorm)) requires [Maven](http://maven.apache.org/). You can install Maven (version 3.x) by following the [Maven installation instructions](http://maven.apache.org/download.cgi).

Navigate to the directory for this tutorial inside the h2o-training repository:

`$ cd h2o-training/tutorials/streaming/storm`  

You should see the following files in this directory:

* storm.md (This document)
* example.R (The R script that builds the GBM model and exports it as a Java POJO)
* training_data.csv (The data used to build the GBM model)
* live_data.csv (The data that predictions are made on; used to feed the spout in the Storm topology)



### 2.2.  Install R

Get the [latest version of R from CRAN](http://www.r-project.org/index.html) and install it on your computer.

### 2.3.  Install the H2O package for R

> Note:  The H2O package for R includes both the R code as well as the main H2O jar file.  This is all you need to run H2O locally on your laptop.

Step 1:  Start R  
`$ R`  

Step 2:  Install H2O from CRAN  
`> install.packages("h2o")`  

> Note:  For convenience, this tutorial was created with the [Markov](http://h2o-release.s3.amazonaws.com/h2o/rel-markov/1/index.html) stable release of H2O (2.8.1.1) from CRAN, as shown above.  Later versions of H2O should also work.

### 2.4.  Development environment

This tutorial was developed with the following software environment.  (Other environments will work, but this is what we used to develop and test this tutorial.)

* MacOS X (Mavericks)
* java version "1.7.0_51" (JDK)
* R 3.1.2
* Storm git hash (insert here)
* curl 7.30.0 (x86_64-apple-darwin13.0) libcurl/7.30.0 SecureTransport zlib/1.2.5


## 3.  A brief discussion of the data

Let's take a look at a small piece of the training_data.csv file for a moment.  This is a synthetic data set created for this tutorial.

`$ head training_data.csv`  

```
"Label","Has4Legs","CoatColor","HairLength","TailLength","EnjoysPlay","StairsOutWindow","HoursSpentNapping","RespondsToCommands","EasilyFrightened","Age", "Noise1", "Noise2", "Noise3", "Noise4", "Noise5"
dog,1,White,0,3,1,0,2,1,1,8,0.207813357003033,0.369364158017561,0.857980248518288,0.609498742735013,0.099459623452276
dog,1,Brown,0,2,1,1,3,1,1,10,0.652755315881222,0.123430503997952,0.63592501427047,0.945240517612547,0.752638266654685
dog,1,Spotted,0,8,1,1,2,1,1,19,0.120261456118897,0.875902567990124,0.748564072418958,0.94046531803906,0.831317493226379
dog,1,White,0,2,1,1,2,1,1,18,0.244165377225727,0.971678811358288,0.00645321514457464,0.0119030212517828,0.0655347942374647
dog,1,Brown,0,1,1,1,2,1,1,17,0.185770921641961,0.423378459410742,0.580749548971653,0.304840496741235,0.135181164368987
dog,1,Brown,0,2,1,1,10,1,1,5,0.602213507518172,0.369238587561995,0.0173729541711509,0.220352463191375,0.723928186809644
dog,1,Brown,0,6,1,1,2,1,1,5,0.518676368519664,0.12482417980209,0.948365441989154,0.177089381730184,0.587333778617904
dog,1,Brown,1,5,1,1,2,1,1,18,0.809266310418025,0.170960413524881,0.753230679314584,0.104151085484773,0.0647049802355468
dog,1,Spotted,0,2,1,1,2,1,1,7,0.710338631412014,0.0687884974759072,0.504301907494664,0.0846393911633641,0.359005510108545
```

Note that the first row in the trailing data set is a header row specifying the column names.

The response column (i.e. the "y" column) we want to make predictions for is Label.  It's a binary column, so we want to build a classification model.  The response column is categorical, and contains two levels, 'cat' and 'dog'.

The remaining columns are all input features (i.e. the "x" columns) we use to predict whether each new observation is a 'cat' or a 'dog'.  The input features are a mix of integer, real, and categorical columns.


## 4.  Using R to build a gbm model in H2O and export it as a Java POJO

<a name="RPOJO"></a>
###  4.1.  Build and export the model

The example.R script builds the model and exports the Java POJO to the generated_model temporary directory.  Run example.R as follows:

`$ R -f example.R`  

```
R version 3.1.2 (2014-10-31) -- "Pumpkin Helmet"
Copyright (C) 2014 The R Foundation for Statistical Computing
Platform: x86_64-apple-darwin13.4.0 (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

  Natural language support but running in an English locale

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

> #
> # Example R code for generating an H2O Scoring POJO.
> #
> 
> # "Safe" system.  Error checks process exit status code.  stop() if it failed.
> safeSystem <- function(x) {
+   print(sprintf("+ CMD: %s", x))
+   res <- system(x)
+   print(res)
+   if (res != 0) {
+     msg <- sprintf("SYSTEM COMMAND FAILED (exit status %d)", res)
+     stop(msg)
+   }
+ }
> 
> library(h2o)
Loading required package: RCurl
Loading required package: bitops
Loading required package: rjson
Loading required package: statmod
Loading required package: survival
Loading required package: splines
Loading required package: tools

----------------------------------------------------------------------

Your next step is to start H2O and get a connection object (named
'localH2O', for example):
    > localH2O = h2o.init()

For H2O package documentation, ask for help:
    > ??h2o

After starting H2O, you can use the Web UI at http://localhost:54321
For more information visit http://docs.0xdata.com

----------------------------------------------------------------------


Attaching package: ‘h2o’

The following objects are masked from ‘package:base’:

    ifelse, max, min, strsplit, sum, tolower, toupper

> 
> cat("Starting H2O\n")
Starting H2O
> myIP <- "localhost"
> myPort <- 54321
> h <- h2o.init(ip = myIP, port = myPort, startH2O = TRUE)

H2O is not running yet, starting it now...

Note:  In case of errors look at the following log files:
    /var/folders/tt/g5d7cr8d3fg84jmb5jr9dlrc0000gn/T//Rtmps9bRj4/h2o_tomk_started_from_r.out
    /var/folders/tt/g5d7cr8d3fg84jmb5jr9dlrc0000gn/T//Rtmps9bRj4/h2o_tomk_started_from_r.err

java version "1.7.0_51"
Java(TM) SE Runtime Environment (build 1.7.0_51-b13)
Java HotSpot(TM) 64-Bit Server VM (build 24.51-b03, mixed mode)


Successfully connected to http://localhost:54321 

R is connected to H2O cluster:
    H2O cluster uptime:         1 seconds 587 milliseconds 
    H2O cluster version:        2.8.1.1 
    H2O cluster name:           H2O_started_from_R 
    H2O cluster total nodes:    1 
    H2O cluster total memory:   3.56 GB 
    H2O cluster total cores:    8 
    H2O cluster allowed cores:  2 
    H2O cluster healthy:        TRUE 

Note:  As started, H2O is limited to the CRAN default of 2 CPUs.
       Shut down and restart H2O as shown below to use all your CPUs.
           > h2o.shutdown(localH2O)
           > localH2O = h2o.init(nthreads = -1)

> 
> cat("Building GBM model\n")
Building GBM model
> df <- h2o.importFile(h, "training_data.csv");
^M  |                                                                            ^M  |                                                                      |   0%^M  |                                                                            ^M  |======================================================================| 100%
> y <- "Label"
> x <- c("NumberOfLegs","CoatColor","HairLength","TailLength","EnjoysPlay","StairsOutWindow","HoursSpentNapping","RespondsToCommands","EasilyFrightened","Age", "Noise1", "Noise2", "Noise3", "Noise4", "Noise5")
> gbm.h2o.fit <- h2o.gbm(data = df, y = y, x = x, n.trees = 10)
^M  |                                                                            ^M  |                                                                      |   0%^M  |                                                                            ^M  |======================================================================| 100%
> 
> cat("Downloading Java prediction model code from H2O\n")
Downloading Java prediction model code from H2O
> model_key <- gbm.h2o.fit@key
> tmpdir_name <- "generated_model"
> cmd <- sprintf("rm -fr %s", tmpdir_name)
> safeSystem(cmd)
[1] "+ CMD: rm -fr generated_model"
[1] 0
> cmd <- sprintf("mkdir %s", tmpdir_name)
> safeSystem(cmd)
[1] "+ CMD: mkdir generated_model"
[1] 0
> cmd <- sprintf("curl -o %s/GBM_generated_model.java http://%s:%d/2/GBMModelView.java?_modelKey=%s", tmpdir_name, myIP, myPort, model_key)
> safeSystem(cmd)
[1] "+ CMD: curl -o generated_model/GBM_generated_model.java http://localhost:54321/2/GBMModelView.java?_modelKey=GBM_9d538f637ef85c78d6e2fea88ad54bc9"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
^M  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0^M100 27041    0 27041    0     0  2253k      0 --:--:-- --:--:-- --:--:-- 2400k
[1] 0
> cmd <- sprintf("curl -o %s/h2o-model.jar http://127.0.0.1:54321/h2o-model.jar", tmpdir_name)
> safeSystem(cmd)
[1] "+ CMD: curl -o generated_model/h2o-model.jar http://127.0.0.1:54321/h2o-model.jar"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
^M  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0^M100  8085  100  8085    0     0  3206k      0 --:--:-- --:--:-- --:--:-- 3947k
[1] 0
> cmd <- sprintf("sed -i '' 's/class %s/class GBM_generated_model/' %s/GBM_generated_model.java", model_key, tmpdir_name)
> safeSystem(cmd)
[1] "+ CMD: sed -i '' 's/class GBM_9d538f637ef85c78d6e2fea88ad54bc9/class GBM_generated_model/' generated_model/GBM_generated_model.java"
[1] 0
> 
> cat("Note: H2O will shut down automatically if it was started by this R script and the script exits\n")
Note: H2O will shut down automatically if it was started by this R script and the script exits
> 
```

### 4.2.  Look at the output

The generated_model directory is created and now contains two files:

`$ ls -l generated_model`  

```
mbp2:storm tomk$ ls -l generated_model/
total 72
-rw-r--r--  1 tomk  staff  27024 Nov 15 16:49 GBM_generated_model.java
-rw-r--r--  1 tomk  staff   8085 Nov 15 16:49 h2o-model.jar
```

The h2o-model.jar file contains the interface definition, and the GBM_generated_model.java file contains the Java code for the POJO model.

The following three sections from the generated model are of special importance.

####  4.2.1.  Class name

```
public class GBM_generated_model extends water.genmodel.GeneratedModel {
```

This is the class to instantiate in the Storm bolt to make predictions.  (Note that we simplified the class name using sed as part of the R script that exported the model.  By default, the class name has a long UUID-style name.)


####  4.2.2.  Predict method

```
public final float[] predict( double[] data, float[] preds)
```

predict() is the method to call to make a single prediction for a new observation.  **_data_** is the input, and **_preds_** is the output.  The return value is just **_preds_**, and can be ignored.

Inputs and Outputs must be numerical.  Categorical columns must be translated into numerical values using the DOMAINS mapping on the way in.  Even if the response is categorical, the result will be numerical.  It can be mapped back to a level string using DOMAINS, if desired.  When the response is categorical, the preds response is structured as follows:

```
preds[0] contains the predicted level number
preds[1] contains the probability that the observation is level0
preds[2] contains the probability that the observation is level1
...
preds[N] contains the probability that the observation is levelN-1

sum(preds[1] ... preds[N]) == 1.0
```

In this specific case, that means:

```
preds[0] contains 0 or 1
preds[1] contains the probability that the observation is ColInfo_15.VALUES[0]
preds[2] contains the probability that the observation is ColInfo_15.VALUES[1]
```


####  4.2.3.  DOMAINS array


```
  // Column domains. The last array contains domain of response column.
  public static final String[][] DOMAINS = new String[][] {
    /* NumberOfLegs */ ColInfo_0.VALUES,
    /* CoatColor */ ColInfo_1.VALUES,
    /* HairLength */ ColInfo_2.VALUES,
    /* TailLength */ ColInfo_3.VALUES,
    /* EnjoysPlay */ ColInfo_4.VALUES,
    /* StairsOutWindow */ ColInfo_5.VALUES,
    /* HoursSpentNapping */ ColInfo_6.VALUES,
    /* RespondsToCommands */ ColInfo_7.VALUES,
    /* EasilyFrightened */ ColInfo_8.VALUES,
    /* Age */ ColInfo_9.VALUES,
    /* Noise1 */ ColInfo_10.VALUES,
    /* Noise2 */ ColInfo_11.VALUES,
    /* Noise3 */ ColInfo_12.VALUES,
    /* Noise4 */ ColInfo_13.VALUES,
    /* Noise5 */ ColInfo_14.VALUES,
    /* Label */ ColInfo_15.VALUES
  };
```

The DOMAINS array contains information about the level names of categorical columns.  Note that Label (the column we are predicting) is the last entry in the DOMAINS array.

<a name="BuildStorm"></a>
## 5.  Building Storm and the bolt for the model

### 5.1 Build storm and import into IntelliJ
To build storm navigate to the cloned repo and install via Maven:

`$ cd storm && mvn clean install -DskipTests=true`  

Once storm is built, open up your favorite IDE to start building the h2o streaming topology. In this tutorial, we will be using [IntelliJ](https://www.jetbrains.com/idea/).

To import the storm project into your IntelliJ please follow these screenshots:

Click on "Import Project" and find the storm repo. Select storm and click "OK"  
![](iJ_1.png)

Import the project from extrenal model using Maven, click "Next"  
![](iJ_2.png)


Ensure that "Import Maven projects automatically" check box is clicked (it's off by default), click "Next"  
![](iJ_3.png)

That's it! Now click through the remaining prompts (Next -> Next -> Finish).

Once inside the project, open up *examples/storm-starter/test/jvm/storm.starter*. Yes, we'll be working out of the test directory.

### 5.2  Build the topology

The topology we've prepared has one spout [TestH2ODataSpout]() and [two bolts]() (a "Score Bolt" and a "Classifier Bolt"). Please copy the pre-built bolts and spout into the *test* directory in IntelliJ. Your project should now look like this:

![](ij_4.png)


## 6.  Copying the generated POJO files into a Storm bolt build environment

We are now ready to import the H2O pieces into the IntelliJ project. We'll need to add the *h2o-model.jar* and the scoring POJO.

To import the *h2o-model.jar* into your IntelliJ project, please follow these screenshots:

File > Project Structure…  
![](iJ_6.png)

Click the "+" to add a new dependency  
![](iJ_7.png)

Click on Jars or directories…  
![](iJ_8.png)

Find the h2o-model.jar that we previously downloaded with the R script in [section 4](#RPOJO)
![](iJ_9.png)

Click "OK"", then "Apply", then "OK".

You now have the h2o-model.jar as a depencny in your project.

We now copy over the POJO from [section 4](#RPOJO) into our storm project. Your project directory should look like this:

![](ij_10.png)

In order to use the GBMPojo class, we add a bolt to our H2OStormStarter wich has the following code:


```
@Override public void execute(Tuple tuple) {
      GBMPojo p = new GBMPojo();  // instantiate a new GBMPojo object

      // get the input tuple as a String[]
      ArrayList<String> vals_string = new ArrayList<String>();
      for (Object v : tuple.getValues()) vals_string.add((String)v);
      String[] raw_data = vals_string.toArray(new String[vals_string.size()]);

      // the score pojo requires a single double[] of input.
      // We handle all of the categorical mapping ourselves
      double data[] = new double[raw_data.length-2];         //drop the Label and ID

      String[] colnames = GBMPojo.NAMES;

      // if the column is a factor column, then look up the value, otherwise put the double
      for (int i = 2; i < raw_data.length; ++i) {
        data[i-2] = p.getDomainValues(colnames[i]) == null
                ? Double.valueOf(raw_data[i])
                : p.mapEnum(p.getColIdx(colnames[i]), raw_data[i]);
      }

      // get the predictions
      float[] preds = new float[GBMPojo.NCLASSES+1]);
      p.predict(data, preds);

      // emit the results: observation ID, expected label, probability of observation being 'dog'
      _collector.emit(tuple, new Values(raw_data[0], raw_data[1], preds[1])); 
      _collector.ack(tuple);
    }
```

## 7.  Running a Storm topology with your model deployed

Finally, we can run the topology by right-clicking on H2OStormStarter and running. Here's a screen shot of what that looks like:

![](ij_11.png)

## 8.  Watching predictions in real-time
![](cats_n_dogs.png)


## References

* [CRAN](http://cran.r-project.org)
* GBM

    [The Elements of Statistical Learning. Vol.1. N.p., page 339](http://www.stanford.edu/~hastie/local.ftp/Springer/OLD//ESLII_print4.pdf)  
    Hastie, Trevor, Robert Tibshirani, and J Jerome H Friedman.  
    Springer New York, 2001.
    
    [Data Science with H2O (GBM)](http://docs.0xdata.com/datascience/gbm.html)
    
    [Gradient Boosting (Wikipedia)](http://en.wikipedia.org/wiki/Gradient_boosting)

* [H2O](http://h2o.ai)
* [H2O Markov stable release](http://h2o-release.s3.amazonaws.com/h2o/rel-markov/1/index.html)
* [Java POJO](http://en.wikipedia.org/wiki/Plain_Old_Java_Object)
* [R](http://www.r-project.org)
* [Storm](https://storm.apache.org)