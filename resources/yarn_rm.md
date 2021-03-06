<!-- CSS work goes here for the time being -->
<!-- set a:link text-decoration to none -->
<!-- set a:hover text-decoration to underline -->
<!-- http://forums.markdownpad.com/discussion/143/include-pdf-pagebreak-instructions-in-markdown/p1 -->

---
<div style="page-break-after: always;"></div>

# <center> <a name="yarn_rm_section"/>YARN & RM

* <a href="#mrv2_review">YARN/MRv2 Design</a>
* <a href="#YARN_overview">What YARN Does</a>
* <a href="#migrating_mrv1_mrv2">Migrating from MRv1 to YARN</a>
* <a href="#RM_overview">Resource Management Overview</a>

---
<div style="page-break-after:always;"></div>

## <center> The MapReduce service: Roles & Pain Points

* JobTracker responsibilities
    * Schedules jobs
    * Monitors TaskTrackers
    * Updates jobs status
    * Caches recent job history
* TaskTrackers declare some number of mapper and/or reducer <i>slots</i> 
    * Getting the slot count right is based on cores, spindles, workload, and experience
    * Fully utilizing cluster resources for execution is problematic
* At ~4k TaskTrackers, it is claimed the JobTracker becomes a bottleneck
* JobTracker history cache can be overwhlemed by many fast-failing jobs

---
<div style="page-break-after: always;"></div>

## <center> <a name="mrv2_review"/> YARN/MRv2: Design

### <center> Graphic overview

<center><img src="http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif"></center>

---
<div style="page-break-after: always;"></div>

## <center> MRv2: Roles

* Resource Manager (RM)
    * Supervises NodeManagers, schedules jobs
    * Default port 8088
* NodeManager
    * Launches/manages containers
    * Updates the Resource Manager 
* A container is a resource-configurable execution context
* Application Master (AM)
    * A container that controls a specific job
    * Example: <code>org.apache.hadoop.mapreduce.v2.app.MRAppMaster</code>
* JobHistory Server
    * Logs job status info from NodeManagers
    * Default port: 19888

---
<div style="page-break-after: always;"></div>

## <center> YARN Job Control (MR)

1. Client submits a job 
2. RM identifies the appropriate AM and loads it
3. AM requests data blocks via the NameNode
4. AM also requests container resources via the RM
5. RM returns a list of available containers to the AM
6. AM requests containers (via their Node Manager) for mapping
7. RM then invokes an auxiliary shuffle service 
8. AM requests containers for reducing tasks
9. The AM releases each container upon task completion and updates the RM 
10. RM updates the Job History server

---
<div style="page-break-after: always;"></div>

## <center> <a name="YARN_overview"/>What YARN Changes

* YARN supercedes the MRv1 JobTracker and TaskTrackers
* Each MRv2 AM instance controls one job 
    * The Application Master (AM) schedules, executes, supervises, and gets resources for its jobs
* YARN = <strong>Y</strong>et <strong>A</strong>nother <strong>R</strong>esource <strong>N</strong>egotiator
    * Abstracts resource controls from the execution framework
    * Allows support for engines such as Spark 
    * The [NextGenMapReduce page](http://wiki.apache.org/hadoop/NextGenMapReduce) offers rationale for YARN

---
<div style="page-break-after: always;"></div>

## <center>YARN over MRv1 or Spark Standalone</center>

* Better resource scalability and utilization
    * Especially for very large clusters
* Better performance for well-known use cases
    * A large cluster running many small jobs
    * Isolating resources for MR and Spark 
* The Holy Grail: one RM controlling all cluster processing
    * RM HA is available
    * Node labeling: dedicating hardware to specific tasks
    * Custom, pluggable classifiers for auditing and reporting

---
<div style="page-break-after: always;"></div>

## <center> <a name="migrating_mrv1_mrv2"/> Migrating from MRv1 to MRv2

* Read <a href="http://blog.cloudera.com/blog/2014/04/apache-hadoop-yarn-avoiding-6-time-consuming-gotchas/">Jeff Bean's blog post on common gotchas</a>, including:
    * Changing from slots to container sizing is just one piece.
    * Getting an [apples-to-apples performance comparison] (http://blog.cloudera.com/blog/2014/02/getting-mapreduce-2-up-to-speed/) is hard
    * Calculating JVM heap requirements is different
    * YARN has to log messages for any potential framework , not just MRv2
        * Log messages are more generic
    * [Too many fast-failing AMs can still bomb the service] (https://issues.apache.org/jira/browse/YARN-1913)

---
<div style="page-break-after: always;"></div>


## <center> <a name="RM_overview"/>Cluster Resource Management

<p><i>sManaging cluster resources for all services is divided into three layers</i></p>

1. <a href="#rm_service_isolation">Service-level isolation</a>
    * Sets minimum resources for all cluster services, including YARN
    * E.g., HDFS, HBase, Impala, Search, MRv1
2. <a href="#admission_control">Using Admission Controls</a>
    * Resource priority based on request, service type
    * Same approach as the Impala service
3. <a href="#dynamic_prioritization">Dynamic prioritization</a>
    * Allocating resources per job by rule

---
<div style="page-break-after: always;"></div>

## <center> <a name="rm_service_isolation"/>Service-level Isolation (cgroups)

* Sets a coarse, minimum percentage of resources per service
    * Percentages are enforced under contention
* Cloudera Manager uses [Linux Control Groups](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CM5/latest/Cloudera-Manager-Managing-Clusters/cm5mc_cgroups.html), so controls are limited by OS support
    * Could support CPU, memory, disk I/O, and network limits, if available
    * *Cluster > ClusterName > Static Service Pools*
    * [A hands-on intro to cgroups](http://riccomini.name/posts/hadoop/2013-06-14-yarn-with-cgroups/)

---
<div style="page-break-after: always;"></div>

## <center> <a name="rm_admission_control"/> Moderating YARN and Impala needs

* [Throttle and queue control for Impala queries only](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/latest/Impala/Installing-and-Using-Impala/ciiu_admission.html)
* Enabled by default starting with Impala 1.3 
* There are three documented models for sharing resources between Impala and YARN 
    1. Define minimum resources for YARN and Impala under contention (Static Service Pools)
    2. YARN defines a pool for Impala; Impala applies Admission Controls
    3. YARN manages Impala through LLAMA (support dropped in CDH 5.6)
* CM supports [Dynamic Resource Pools](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CM5/latest/Cloudera-Manager-Managing-Clusters/cm5mc_resource_pools.html)
    * A <i>configuration sets</i> is used to define a client group (e.g., prod, mktg, batch, queries)
    * Some <i>scheduling rules</i> inform the configuration set's policy
    * Pool resources may be governed by user permissions, query count, queue size, memory demand
* In the CM UI, see *Cluster > ClusterName > Dynamic Resource Pools*

---
<div style="page-break-after: always;"></div>

## <center> More on Admission Control </center>

* Impala and YARN Uses the same pool definitions 
* Three decisions: execute, queue, or reject a query
* Decision factors:
    * Currently running queries
    * Memory available
    * Current queue length
* The local <code>impalad</code> decides based cached information about the cluster
    * To compensate for potentially stale resource data, admission control is soft
* Impala favors running one too many tasks over preserving headroom
    * Work to improve this decision-making is ongoing

---
<div style="page-break-after: always;"></div>

## <center> <a name="rm_dynamic_prioritization"/> A Word on Dynamic Prioritization 

* <strong>L</strong>ow-<strong>L</strong>atency <strong>A</strong>pplication <strong>MA</strong>ster ([LLAMA](http://cloudera.github.io/llama/)) for Impala
    * Released with CDH5 as a beta component
    * Concept: run all Impala queries through one AM
        * Interaction with the RM is not NRT-friendly
    * Project dropped after C5.5
* The objectives are the same
    * Balance low-latency queries with batch processing
    * Find more efficient means to dispatch multiple scheduler queues
    * Devise opportunistic processing schemes for more efficient utilization
    * Improve resource estimation (e.g., Impala's <code>COMPUTE STATS</code>)

---
<div style="page-break-after: always;"></div>

## <center> Current Practices for YARN & Impala

* Keep alert to best practice updates
    * Set and monitor Static Resource Pools to manage contention
    * Use Admission Control with Impala to specify priorities and enforce limits
* Advise customers the LLAMA experiment is over

---
<div style="page-break-after: always;"></div>

## <center> YARN/RM Lab: Doing the Math

* [Use the documentation]
http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_yarn_tuning.html
(http://www.cloudera.com/content/cloudera/en/documentation/core/v5-3-x/topics/cdh_ig_yarn_tuning.html)

 to answer the following:
    * How much RAM should be allocated per <code>impalad</code> process, at least?
    * What percentage of RAM should be earmarked for the OS and system services?
    * Under what condition(s) would you factor two cores per spindle into your calculations?
    * Assume a cluster with 20 vcores, 128 GB RAM, and 10 spindles on each of eight worker nodes.
        * Using the Container Formula given in Table 8, calculate an appropriate value for <code>mapreduce.jobs.maps</code>
        * List each factor and value leading to your result

Add your answers to the file <code>resources/0_YarnCalcs.md</code> in your repository

---
<div style="page-break-after: always;"></div>

## <center> YARN/RM Lab: Static Service Pools

* In CM, navigate by menu: Clusters->YARN Applications or by service: YARN(MR2 Included) -> Applications
    * Click the Charts tab and capture the page.
* Navigate to Clusters -> Static Service Pools
    * Allocate 25% to HDFS and 75% to YARN and click Continue
    * On the Step 2 of 4 page, review the sections and proposed values
    * Complete the wizard: redeploy client configurations and restart the cluster
    * Capture the Step 1 of 4 page after the restart
* Add the screenshot to your repo with the file name <code>resources/1_static_pools.png</code>

---
<div style="page-break-after: always;"></div>

## <center> YARN/RM Lab: Tuning for YARN

* Now you can calculate YARN settings for your EC2 cluster and test them

* Review the file <code>resources/YARNtest.sh</code> in your course repository
* Copy this script to any non-worker node in your cluster
    * Add code to print out the default value of each job property mentioned in the script
    * Set the loop parameters to reflect your calculations along with higher and lower values
    * Add the <code>time(1)</code> command to show duration for each job
* Run the script 
    * Your highest value can max out the cluster to demonstrate resource enforcement 
* Add the following to your repository's <code>resources/</code> directory
    * The completed version of your script (overwrite the initial one)
    * The output of your slowest run in <code>2_YARNslow.md</code>
    * The output of your fastest run in <code>3_YARNfast.md</code>
* Email the instructors when you have completed these labs
