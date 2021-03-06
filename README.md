# KnowledgeStore v1.8.1 Docker Container
## READ CAREFULLY BEFORE STARTING

### 0. Background on Docker

The Docker daemon uses Linux-specific kernel features, and runs natively in linux based systems. On OS X and Windows, docker will be run inside a linux-based Virtual Machine, therefore some changes on the settings of the Virtual Machine may be needed to adjust the memory and disk space allocated based on the quantity of data to be populated into the KS. For 20K documents, we suggest to allocate at least 16GB of RAM and 20GB of disk.

RAM-wise, this KnowledgeStore is pre-configured so that Virtuoso can use 12GB of RAM. This configuration can be changed in the **`data/instances/ksdemo/etc/virtuoso.ini`** file (see _NumberOfBuffers_ and _MaxDirtyBuffers_ variables. Check the Virtuoso documentation for more details.)

### 1. Preparation and build
Download the whole content of this directory. Besides, the dockerfile, it contains a few scripts to build / run the docker container, and a **`data`** folder (shared with the container), used to store the KnowledgeStore configuration files, DBs, useful scripts, etc.

To build the KnowledgeStore container, simply run **`./build_ks_container.sh`**.
This will create a Docker image named **`ks`**, based on the **`centos`** operating system image. These images will take around 1.1GB of disk space. You can remove them by calling **`docker rmi ks`** and then **`docker rmi centos:7.2.1511`** (make sure to remove any container based on these images, first - see below). 

You will be able to access the docker container via ssh using some predefined credentials (user: root; password: password). Feel free to define more secure credentials editing the line `RUN echo 'root:password'` in the Dockerfile, replacing `password` with any string of your choice before runnning the **`./build_ks_container.sh`** script.

### 2. Starting the container

To start the container, simply run **`./start_ks_container.sh`**. This script will create (if it does not exist) and run a docker container named **`ks-demo`** based on the **`ks`** image, sharing some of the folders under **`data`**, and exposing some the KnowledgeStore services on host ports 50022 (sshd), 50051 (virtuoso), and 50053 (knowledgestore). If these ports are closed on the host system, change them as appropriate by modifying the script.

You can check if the KnowledgeStore is properly running by accessing its User Interface at **`http://localhost:50053/ui`**.

You can stop the container by issuing **`docker stop ks-demo`**. To start again the container, call **`docker start ks-demo`**. To remove the container (once stopped), call **`docker rm ks-demo`**.


### 3. Logging-in into the docker container

To login into the **`ks-demo`** running container, type **`ssh root@localhost -p 50022`** (root password is set to _password_ in the Dockerfile).

Within the docker container, you can do several tasks related to the KnowledgeStore, including:

* NAF population
* RDF population
* star/stop/restart/clean the KnowledgeStore 
* compute statistics
* run RDFpro to process RDF files

Next we describe a typical NewsReader (http://www.newsreader-project.eu) KnowledgeStore population cycle.

### 4. Populating the KnowledgeStore (Newsreader version)

First of all, you have to populate the RDF background knowledge, which can be downloaded from here:

* DBpedia 2015.4 (size: 1.3Gb): http://knowledgestore.fbk.eu/files/vm/bk/dbpedia2015en_allForKS.tql.gz
* ESO v2.0 (size: 40Kb) : http://knowledgestore.fbk.eu/files/vm/bk/ESO_v2_all_forKS.tql.gz

We suggest to move these files in the **`data/additional_data`** subfolder.

Then, after logging into the container, run **`populate_tql_gz.sh /data/additional_data/dbpedia2015en_allForKS.tql.gz`** (if different, change the path of dbpedia2015en_allForKS.tql.gz appropriately). The population will start (wait for the script to terminate). Depending on the hardware, time may vary (on an average specs desktop, it may take around 15 minutes).
You can watch the number of triples in the knowledgestore _growing_ by periodically accessing **`http://localhost:50053/ui?action=sparql&query=SELECT+%28COUNT%28*%29+AS+%3Fn%29+%7B+%3Fs+%3Fp+%3Fo++%7D&timeout=`**, which basically fires a count triples query.
When done, the Virtuoso component of the KnowledgeStore will be automatically restarted. Wait for the KnowledgeStore to be up again (e.g., run again the count query) before any further activity.

Similarly, populate the ESO ontology by running **`populate_tql_gz.sh /data/additional_data/ESO_v2_all_forKS.tql.gz`** (same considerations as for the DBpedia population). Should end in few seconds (~30 seconds).


Now, the KnowledgeStore is ready to be populated with NAFs and RDF triples extracted from text. [At this point, it may be good make a backup copy of the virtuoso DB. This copy can be used to avoid the re-population from scratch of the background knowledge content when populating the KnowledgeStore with new content. First, stop the KnowledgeStore running the script **`stop_ks.sh`**. Then, just copy (and maybe gzip) with **`cp`** the file in **``data/instances/ksdemo/var/db/virtuoso.db`**. When done, start again the KnowledgeStore with **`start_ks.sh`**]

We show the population of content NAFs and RDF with a concrete example, populating the KnowledgeStore with the English Wikinews NAFs and RDF triples.

First of all, download the required files and place them in the **`data/additional_data`** subfolder:

* NAFs (size: 754Mb - 19,775 NAF files): http://knowledgestore.fbk.eu/files/vm/wikinews/NAFs.tar.gz
* RDF (size: 353Mb - ~32M quads): http://knowledgestore.fbk.eu/files/vm/wikinews/RDF.trig.gz (here we assume to have a single trig file. If not, you can build a single trig file from a tar.gz archive containing many trig files -potentially stored in nested subfolders- with a command like **`tar -O -xf original_trigs.tgz | pigz -9 - > RDF.trig.gz`**)

Extract the NAFs files in a separate folder:
- **`mkdir -p NAFs && tar -zxf NAFs.tar.gz -C NAFs/`**

To start the NAF population, type **`populate_NAFs.sh /data/additional_data/NAFs/`**.
You can periodically check the population status by accessing **`http://localhost:50053/resources/count?condition=rdf%3atype%3d%5cnwr%3aNAFDocument`**, which basically counts the number of NAF files contained in the KnowledgeStore.
At the end of the NAF population (on an average specs machine, it should take under 2 hours), the KnowledgeStore is automatically restarted.

Before populating the RDF trig file, some processing (e.g., deduplication, ESO reasoning) has to be performed. Invoke the script **`prepareRDF4Virtuoso.sh RDF.trig.gz all.tql.gz`**. At the end, you will get a gzipped tql file ready to be uploaded into the KnowledgeStore as follow: **`populate_tql_gz.sh all.tql.gz`**. Again, you can check the population status by firing the count triples query above. Once done, the KnowledgeStore is automatically restarted.

To compute statistics on the populated KnowledgeStore, you can run the script **`mkdir -p /data/additional_data/stats/ && computeStats.sh /data/additional_data/stats/`**. The resulting statistics will be created in folder **`/data/additional_data/stats/`**.

To wipe all the data in a populated KnowledgeStore, simply run the script **`stop_and_clean_ks.sh`**. Once done, restart the KnowledgeStore with **`start_ks.sh`**.
