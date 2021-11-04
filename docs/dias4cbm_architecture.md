# JRC CbM system architecture

The objective of these pages, as part of the [JRC CbM GENERAL DOCUMENTATION](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_intro.htm), is to provide an overview on **logic, concepts and elements of the JRC CbM architecture** to all the experts of the Paying Agencies (system administrators, analysts, decision makers, project managers and consultants). The aim is to help them to understand the approach and the software solutions proposed and assess and plan its operational implementation. The technical details of JRC CbM are discussed in the [TECHNICAL DOCUMENTATION](https://jrc-cbm.readthedocs.io/) and the code is made available in the [JRC CbM GitHub space](https://github.com/ec-jrc/cbm/).  

In the first section (CbM system requirements) of this CbM architecture overview, we analyse the requirements of a CbM system. In the second section (JRC CbM implementation), we describe the software platform that we created to implement them. In the last section (JRC CbM roles), we provide additional information on the different roles in the system. The numbers in the table of contents correspond to the marks in [Figure 1](#figure1) (Section 1) and [Figure 2](#figure2) (section 2).

### Table of contents

* [CbM system requirements](#cbm-system-requirements)
  * [Scope of the CbM system](#scope-of-the-cbm-system)  
  * [1. Cloud resources](#cloud-resources)  
  * [2. Sentinel images repository](#sentinel-images-repository)
  * [3. Data repository](#data-repository)
  * [4. Extraction of statistics](#extraction-of-statistics)
  * [5. Data access](#data-access)
  * [6. Interfaces for data use](#interfaces-for-data-use)
* [JRC CbM implementation](#jrc-cbm-implementation)
  * [Overview](#overview)
  * [1. DIAS](#dias)  
  * [2. Object storage](#object-storage)
  * [3. PostgreSQL and PostGIS](#postgresql-and-postgis)
  * [4. Python](#python)
  * [5. RESTful API](#restful-api)
  * [6. Jupiter Notebook](#jupiter-notebook)
* [JRC CbM roles](#jrc-cbm-roles)  

## CbM system requirements

The main elements of a CbM system are illustrated in [Figure 1](#figure1). This architecture is modular and reflects the general requirements of any CbM system. The individual elements are described in the following sub-sections.  
![](/docs/img/cbm_dias_structure.png)  
![](/img/cbm_dias_structure.png)  
<a name="figure1">Figure 1.</a> Main elements of a CbM system   

### Scope of the CbM system  
The scope of the CbM system is to exploit the time series of Sentinel data to continuously monitoring all the agricultural parcels, i.e. the polygons from the Land Parcel Identification System (LPIS) and  Geospatial Aid Application (GSAA) that are associated with a CAP scheme. It generates the minimum required information to confirm/reject compliance with the declared practice and to communicate discrepancies to the farmers in real time, so that they can be amended in due time, if needed.  
This process is optimized through automation and **reduction of information** to handle the data load (i.e. summarize the spatio-temporal image stack of Sentinel into time series of bands statistics per agricultural parcel). The results feed into a traffic light system for follow up: most of the cases will have a definite classification (green: confirmed, red: rejected), while some of them will require further investigation (yellow: inconclusive). This reduces considerably the situations to be verified with other tools (e.g. geotagged photos, higher resolution satellite images, orthophotos, field visits).  
The JRC CbM system is based on two architecture layers: the **backend** and the **frontend** (in [Figure 2](#figure2), backend and frontend are identified by the blue and the orange boxes). The backend server provides the end-points to retrieve data: it includes the physical infrastructure and the routines that generates the information used by the analysts and the decision makers. It is developed, managed and maintained by a system administrator with expertise in cloud computing and big data analytics. An extended introduction to this layer is illustrated in the [SYSTEM DEVELOPMENT](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_setup.html) documentation page. The frontend is the component manipulated by the user: it provides access to the data generated by the backend through standard Application Programming Interface (API). It includes the interfaces and the code for using the data to support typical Paying Agencies functions. Frontend users do not require backend expertise. More information are available in the [DATA USE](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_use.html) documentation page and in the [DATA ANALYSIS](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_analysis.html) documentation page.  
In the JRC CbM system the boundary between the two layers is fluid. Once new procedures are defined and tested by analysts in the frontend, they can be integrated into the backend and automatically available to users as starting point for additional processing (e.g. analysis, reporting, classification).  

The elements of the system are described below.  

### Cloud resources
The huge **amount of data** (Sentinel-1 and Sentinel-2, parcels) to be stored and the **resources needed to process them** require an hardware infrastructure that can be hardly set up on a local system. Cloud platforms offer the possibility to optimize the hardware resources according to the specific needs and scale to large data volume without the burden of extensive hardware and software maintenance. In addition, direct access to Sentinel data is needed to avoid downloading the images on a local system. Keeping all the data in the same (cloud-based) container (e.g. images, parcels, statistics) is a further advantage for the efficiency of the CbM process and enforce data consistency and accessibility. Using cloud resources, it is possible to create modular processing pipelines that are tailored to the specific needs of the CbM and the technical capacities of the end users.  

### Sentinel images repository
Sentinel-1 and Sentinel-2 data are at the heart of the system. The information must be accessible in near real time and for all the agricultural areas under assessment. The huge amount of data, tens of petabytes (i.e. ten million gigabytes), requires **dedicated and performant tools for storage and access**, being a old-fashion flat-file based approach inefficient and not practical in this context. Sentinel imageries should be available to end users as Application (or Analysis) Ready Data (ARD), i.e. in a format that is ready for analysis so that users can work with the images without the burden of complex and time consuming pre-processing steps. Copernicus ARD (CARD) should, as a minimum, include georeferenced, calibrated sensor data (Level 2).  

### Data repository
In addition to Sentinel images, the system has to manage and process other control data sets in order to check farmers' declaration.  
The first one is the **agricultural parcels**: polygons from the Land Parcel Identification System (LPIS) and Geospatial Aid Application (GSAA) that are associated with a CAP scheme, managed by the PA. It is a spatial layer with associated non-spatial attributes. The number of polygons can be in the order of hundreds of thousands.  
The second layer is the **metadata of Sentinel images**. This data set is derived from the Sentinel archive and includes the key information of each image (e.g. id, date, extent) and make images retrieval and identification faster.  
Finally, the system has to store the **signatures** of Sentinel bands extracted per polygon (also called *reduction* in the context of JRC CbM).  
The spatial nature of these data sets, their well defined relations, their size and the need for scalability (data sets increase quickly over time), the importance of data consistency and the need to make the data available to multiple tools and users point to spatial relational databases (SRDBMS) as the state of art technical solution.  
In an operational national CbM system, a relational database can be used not only to support extraction and management of signatures data, but is also a good candidate as central repository for all the information relevant for the CAP process. In such a context, the technical solution depends on the specific goals and constraints of each Paying Agency (parcel classification according to the traffic light sysyem is included in [Figure 1](#figure1) with transparency as it is just an option for an extension of the CbM).  

### Extraction of statistics
The key information to assess compliance of a declared practice is the **time series of statistics** (mean, max, min, median, 25 quantile, 75 quantile, standard deviation), also called signatures, of Sentinel bands for each polygon. It is generated with a **spatial intersection** between the Sentinel-1 and Sentinel-2 relevant bands and the polygons from the Land Parcel Identification System (LPIS) and Geospatial Aid Application (GSAA). The most relevant bands for CbM are Sentinel-2 B02, B03, B04, B08, B5, B11 and Sentinel-1 VV, VH. Sentinel-2 scene classification map (SCL) is also important to detect (and ignore) images where the signal is affected by clouds. The image is selected using the image metadata of the Sentinel archive. The availability of Sentinel CARD ensures that the spectral and temporal information (i.e. time series) can be extracted consistently. Although it is technically feasible to generate a CARD time series on demand, it is often convenient to extract large sets of parcels, for pre-selected bands, in a batch process. Cloud infrastructures, and particularly DIAS, are specifically tailored to these kind of tasks, since extraction is typically a massively parallel processing step that can benefit from the use of multiple machines. Technical details are given in the [documentation page on parcel extraction](https://jrc-cbm.readthedocs.io/en/latest/parcel_extraction.html).  
Since CbM covers 100% of the territory, it needs to be supported by smart analytics that filter out the anomalies quickly and consistently. The signal statistics can be used in such analytics, e.g. to support outlier identification, detection of heterogeneity, marker generation. The extraction has to be an automated and performant process given the amount of elements to be calculated. To give an order of magnitude, if the parcels are 500,000, the Sentinel-2 images are 73 (an image every 5 days in a single year) and the bands are 8, the number of signature records generated is about 300 millions.  
Other ancillary data may be required to explain signal artefacts (DEM, weather parameters, etc) and the system should be able to manage them as well.  

### Data access  
The frontend users need to **access the data generated by the backend** (i.e. CARD data, if not already in DIAS archive, and parcel time series of Sentinel spatio-temporal image stacks) to feed into post-processing. While direct access to tables, spatial features and images stored and managed in the cloud infrastructure is possible and recommended for developers/analytics, it exposes the system to multiple issues. First of all, it requires advanced technical skills that analysts and decision makers do not always have in their background (for example SQL, the database language). Then, directly open the system to many users exposes it to potential security risks. Finally, inexperienced users could try to run processing that is not optimized and thus consuming abnormal resources. An intermediate layer that offers pre-defined functionalities and formats to consume data from the backend, for example the time series of statistics and images for a user-defined parcel, can help to address these problems.  
Analytical algorithms and markers are designed and tested in the frontend but when consolidated the system should allow their implementation in the backend.  

### Interfaces for data use  
In addition to an intermediate layer to access the CbM data, analysts and final users require **interactive interfaces** to visualize and post-process the information generated by the system. These interfaces must be coherent with their technical skills and goals. Analysts should be able to interact with data using the most common geospatial processing languages and libraries, while final users need visual output that can be interpreted (e.g. graphs, images, reports) and tools to support the compliance verification procedure.  

## JRC CbM implementation
The JRC CbM implements the architecture shown in [Figure 1](#figure1) according to the requirements discussed in the previous section. It is a tailored solution based on the specific set of tools indicated in [Figure 2](#figure2) and described below. Software modules selected are all open source and make use of open standards. The code created by JRC is open source and available in the [JRC CbM GitHub repository](https://github.com/ec-jrc/cbm/).  
The same structure can be operationalized using other software with similar functionalities if the system has to be integrated in an already existing infrastructure. JRC CbM is designed on a cloud-centric basis, but can also be adapted to run stand-alone.  
![](/docs/img/cbm_dias_software.png)
![](/img/cbm_dias_software.png)  
 <a name="figure2">Figure 2.</a> JRC CbM software platform

### Overview
In the JRC CbM, satellite data are made available in the Object Storage of the [Copernicus Data and Information Access Services (DIAS)](https://www.copernicus.eu/en/access-data/dias) infrastructure and processed in that environment by Python-based modules. The base layers are stored in a spatial database based on PostgreSQL/PostGIS installed in the DIAS server, inside the same environment of the satellite image archive. The server is set up and managed using Ubuntu, Docker and OpenStack. Data access is filtered by a RESTful API that feed the user's interface, based on Jupiter Notebook and Voilà.  

The elements of JRC CbM are described in details below.  

### DIAS

#### What are DIAS
DIAS are five European core computing and storage infrastructure (CREODIAS, MUNDI, ONDA, SOBLOO, Wekeo) provided as Infrastructures as a Service (IaaS). The actual DIAS use by end users, such as MS Paying Agencies, or any other public or private user, is managed through accounts, i.e. subscription services that are costed on the basis of actual use of computing resources (e.g. CPU time, data transfer, additional storage requirements).  DIAS may be used at increasing levels of complexity, e.g. (1) simple CARD downloads, (2) extracting reductions to (3) full data analytics.  
DIAS fit current and future CbM system needs. In addition to the typical features of cloud infrastructures (e.g. configurable compute resources that can scale to needs and possibility to run parallel tasks when tasks can be parallelized, which is the case for CbM), they grant access to a consistent, complete Sentinel data archive and provision of on-demand standard CARD processing. They are based on open industry standards and built up with core open source modules.  
More information on DIAS setup and configuration are reported in the [DIAS setup documentation page](https://jrc-cbm.readthedocs.io/en/latest/setup_prerequisites.html ).

#### DIAS onboarding for MS Paying Agencies
In the context of CAP CbM, DG AGRI together with DG DEFIS (the Copernicus program managers, previously known as DG GROW) have decided to fund the DIAS onboarding of MS Paying Agencies in 2019 and 2020 through ESA work orders. In 2019, the work orders were structured in 3 phases: (1) technical readiness review, (2) onboarding MS Paying Agency on one of the DIAS instances, and (3) operational use of the DIAS in the 2019 CAP CbM. In 2020, the first 2 phases are no longer required, and, based on the experience built up in 2019, onboarding MS can jump directly into the operational phase. The work orders are supported, on a technical level, by DG JRC with the definition of high level requirements, technical adequacy review, methodological assistance for the generation of CARD (see next section) and the definition and initial demonstration of CAP CbM relevant functionalities. 2021+ arrangement is under design: it will depend on decisions on DIAS future and on the new CAP directions.  

#### Outreach
The Outreach activity offers to the Member State willing to take up initiative on detection of agricultural phenomena with Sentinel data the possibility to use an JRC CbM environment set up by GTCAP that takes care of the backend. Here final users and analysts can test the tools and the code with technical support before they create their own CbM system.  

#### DIAS Virtual Machines
The DIAS Virtual Machines (VMs) are managed using Linux as operating system ([Ubuntu](https://ubuntu.com/)). Linux bash scripting is used for orchestration and parsing. Reprojection and other basic spatial processing are based on the [GDAL library](https://gdal.org/). The DIAS tenant can select and configure VMs for specific functions:  
* permanent VMs (e.g. data base server, Jupyter Hub, RESTful)
* transient VMs (use on demand, run large tasks in parallel, tear down)

As a first step, we use Openstack resources "marshalling". [OpenStack](https://www.openstack.org/) is a free, open standard cloud computing platform. It is deployed as infrastructure as a service where virtual servers and other resources are made available to users. It scales horizontally and is designed to scale on hardware without specific requirements.  
In a second step, we "orchestrate" the resources to execute parallel tasks (for example parcel and chip extraction). We use Docker containerization to ease cross-VM installation. [Docker](https://www.docker.com/) is an open platform for developing, shipping, and running applications. It uses OS-level virtualization to deliver software in packages called containers. Containers are isolated from one another and bundle their own software, libraries and configuration files. They can communicate with each other through well-defined channels. Docker enables to separate applications from the infrastructure so that the infrastructure can be managed in the same ways you manage the applications. Docker containers behave like specialized VMs. We use Docker Swarm as container orchestration tool, meaning that it allows us to manage multiple containers deployed across multiple host machines. Figure 3 shows the Copernicus DIAS IaaS architecture.  
![](/docs/img/DIAS_IaaS.png)
![](/img/DIAS_IaaS.png)  
Figure 3. Copernicus DIAS Infrastructures as a Service (*click on the image to enlarge*).  

### Object storage  
#### DIAS object storage  
The best tool to store and manage immutable "Big Data" blobs that are write once and then read often (e.g. YouTube video, DIAS Sentinel data) is object storage. Object storage (also known as object-based storage) is a data storage architecture that manages data as objects, as opposed to other storage architectures like file systems which manages data as a file hierarchy, and block storage which manages data as blocks within sectors and tracks. Each object typically includes the data itself, a variable amount of metadata, and a globally unique identifier. It is simpler to manage and extend than file or block storage, and it is much cheaper. The access is generally slower, especially since it is not optimized for partial reads.  
It requires API to transfer to classical file system to be handle as normal file. CREODIAS, MUNDI, SOBLOO and Wekeo all use [S3](https://en.wikipedia.org/wiki/Amazon_S3) (AWS, GCS standard) object storage. ONDA uses [ENS](https://www.onda-dias.eu/cms/services/catalogues/advanced-api/) (OpenStack Swift).  
Object storages can manage tens of Petabytes data (e.g. CREODIAS ~ 20 PB, Goggle Earth Engine ~ 85 PB).  
In JRC CbM, DIAS object storage is accessed through the Python library [Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html).

#### Copernicus ARD generation
DIAS instances offer a Processing as a Service (PaaS) solution for generating Copernicus Application Ready Data (ARD). ARD is stored in the S3 object store of the DIAS, in public or private buckets.  
Sentinel-1 and Sentinel-2 data are, by default, delivered by ESA through the Copernicus Hub as Level 1 data. Level 2 (atmospherically corrected) Sentinel-2 data is also made available. For Sentinel-1, this is not the case (an anomaly that may be resolved in the future).  
Sentinel-1 ARD processing is done by JRC CbM backend to generate:  
* calibrated geocoded backscattering coefficients from Level 1 GRD (CARD-BS)  
* geocoded coherence from Level 1 SLC S1A and S1B pairs (CARD-COH6)  

If ARD does not already exist in DIAS archive for a given country, it has to be generated by the system. After a bulk generation of ARD is done until the last available date, a backend batch process will ingest (with a typical delay of 1-2 days) the new images that are made available in the ESA hub and transferred to the DIAS.  
ARD generation process is not yet part of JRC CbM documentation, but it is available on request. More information are reported in the [JRC CbM backend section](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_setup.html).  

### PostgreSQL and PostGIS
In the JRC CbM system, the open-source Relational DataBase Management System (RDBMS) [PostgreSQL](https://www.postgresql.org/) with its spatial extension [PostGIS](http://postgis.refractions.net/) is used as data repository. It represents the state of art S(patial)RDBMS, which can efficiently manage very large spatial (and temporal) and non spatial datasets with a complex structure in a multi-user and secure environment.  
In JRC CbM, Sentinel metadata are parsed into PostgreSQL/PostGIS dias_catalogue table, while data on parcels are ingested by the users. Signature data are generated by the JRC CbM backend processing.  
JRC CbM architecture integrates the database into the DIAS server, in the same environment of the object storage that manage Sentinel data, but it can also be deployed on a local server.  
The information about the database content, structure, description, connection, management and how to retrieve data is provided in the [JRC CbM database documentation page](https://jrc-cbm.readthedocs.io/en/latest/setup_database.html).

### Python
The backend processing of Sentinel data extraction to reduce the spatio-temporal image stacks of ARD to parcel time series is based on [Python](https://www.python.org/), a general-purpose programming language with simple syntax and code readability that make coding easy and efficient. It has a large set of specific libraries, particularly for data science.  
Extraction is set up as an automated process that:  
* finds the oldest image that is not yet processed  
* transfers the image bands from the S3 store onto local disk  
* queries the database for all parcels within the image bounds  
* extracts the statistics (μ, σ, min, max, p25, p50, p75) for the bands of the image  
* stores the results in the time series database table  
* clears the local disk  

In principle, there is no reason why parcel extraction could not be offered as a PaaS, and this might be an option for the future, simplifying CbM systems.  

In the JRC CbM, Python is used not only for data extraction but is the syntactic glue of the whole system, especially in the frontend. The [numpy](https://numpy.org/), [pandas](https://pandas.pydata.org/), [geopandas](https://geopandas.org/), [rasterio](https://rasterio.readthedocs.io/), and [osgeo](https://www.osgeo.org/) packages are widely used in many of the system modules. The [psycopg2](https://pypi.org/project/psycopg2/) library is used to connect with the database and retrieve and manipulate the data.  
In a complete CbM system, the backend can also use Python to detect agricultural practices linked to farmer's declaration processing. The output then should feed into traffic light system for follow up (e.g. the need for complementary information).  

### RESTful API  
The analysts and final users (decision makers) need to access the information stored/generated in the backend on the DIAS server, namely 1) parcels, satellite images signature (statistics) time series and image metadata stored in the database tables, and 2) Sentinel CARD available in the S3 store.  
Direct access to a DIAS account that has the credentials to access the various data sets is possible. Data can then be retrieved and visualized with specific tools, e.g. [Python Jupiter Notebooks](https://jupyter.org/) (see next section) and database clients like [PgAdmin](https://www.pgadmin.org/) or [QGIS](https://www.qgis.org/). Nevertheless, we recommend to limit direct DB access to the data to specific cases (i.e. system administrators, developers, users with advanced skills in SQL and good knowledge of the database structure). These users can also generate new data that are stored back in the server and made available to final users (e.g. applying machine learning algorithms to the extracted time series). In all the other cases, the DIAS platform can provide access to data via server interfaces. The final users can use these intermediate layers that provide predefined functionalities to extract data based on a limited and controlled set of parameters. This ensures performance and security by preventing poorly designed resource-intensive image retrieval and database queries and facilitates access to basic users with no knowledge of tools for data extraction (e.g. SQL) as it can serve standard queries and abstracts more complex tasks.  

In the JRC CbM system, the frontend access the backend data sets though a RESTful API. A RESTful API is an application programming interface (API) that conforms to the constraints of REST architectural style. An [API](https://en.wikipedia.org/wiki/API) is a set of definitions and protocols for communicating what you want to that system so it can understand and fulfil the request (sse Figure 4). [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) stands for representational state transfer. It is an architectural style for distributed hypermedia systems: it defines a set of constraints for how the architecture of an Internet-scale distributed hypermedia system should behave. An essential character of the RESTful services is that they follow standard calling conventions, which allow their use both in interactive testing and scriptable machine access. RESTful services are further described in [the documentation pages on RESTful service use](https://jrc-cbm.readthedocs.io/en/latest/api_ts.html) and subsequent pages.  
![](/docs/img/rest_api.png)
![](/img/rest_api.png)  
Figure 4. Schema of the interaction between users and the database through the RESTful API  

In the JRC CbM system, the RESTful API provides:  
* abstracted access to database tables  
* abstracted access to sub-selections of S3 stored CARD data  
* abstracted access to advanced server-side processing routines  

Examples of RESTful requests in JRC CbM are:  
* basic information retrieval on parcels (parcelByLocation)  
* parcel time series statistics (parcelTimeSeries, parcelPeers) [fast]  
* image chip selection, for visualization (chipByLocation, backgroundByLocation (Google, Bing and orthophotos via WMTS)) [slow]  
* image chip selection, full resolution GeoTIFFs (rawChipByLocation, rawChipsBatch, rawS1ChipsBatch)  

RESTful can be integrated in scripts, automated reports, Jupyter notebooks via Python requests.  

Another option for accessing data is the use of a [Jupyter Hub](https://jupyterhub.readthedocs.io/en/stable/) server configured to hide the intrinsic access protocols and supporting interactive scripting and data visualization. A mixture of direct access and RESTful services is also possible, for example with a Jupyter notebook running on the Jupyter Hub that uses both.  
Finally, an alternative use pattern would be to transfer the time series database to a dedicated server set up outside the DIAS cloud infrastructure, for instance, into the PA's ICT infrastructure. RESTful services running on DIAS can then be used to maintain access to the DIAS CARD data store to support on demand generation of image fragments, for instance.  

### Jupiter Notebook
Users can access data directly or through the RESTful API. In both cases, they need an interface to visualize (as graphs, maps, tables) and analyse (applying for example statistical algorithms, machine learning) data according to their needs. In The JRC CbM system, the main tool used for this task is Jupyter Notebooks. [Jupyter Notebooks](https://jupyter.org/) are documents that contain live code (particularly Python), equations, graphics and narrative text.
The open-source interface offers a web-based interactive computational environment for, among the others, data cleaning and transformation, numerical simulation, statistical modelling, data visualization, and machine learning.  
Python is used to get, process and display the data.  
In particular, analysts are interested in the identification of markers. Markers are sequence of signatures (statistics derived from Sentinel bands for each parcels) that can be associated to the agricultural practices declared by the farmer according to CAP schemes (when Sentinel data can pick up the signature of the events and patterns that relate to the practice). Jupiter Notebooks offer an ideal environment to explore and test new markers and analytical approaches.  
Once established by analysts and consolidated, markers can be integrated in the backend and used to confirm/reject compliance with the declared practice, feeding into traffic light system for follow up (e.g. the need for complementary information) by decision makers.  
Final users does not necessarily require an advanced analytical environment offering a wide range of functionalities to research new solutions, but rather need a simple interface for visualization of data generated in the backend for reporting. In this case, we use the [Voilà](https://voila.readthedocs.io/) that converts a Jupyter Notebook into an interactive dashboard. Final users can play with the data through a graphical user interface with no need to code.

## JRC CbM roles
JRC CbM considers roles. Not all roles need to work with all modules. The are thee main classes of roles:  

* ICT experts create and maintain the infrastructures with its required server components, e.g. monitoring CARD production, run extractions. This backend role can be managed by one person per MS (or even DIAS).  
* Data analytics experts (frontend) program and run core analytics (e.g. extraction, machine learning, markers). They can be both internal and external to the PA.  
* Data consumers (frontend) extract, cross-check, verify, combine, decide and report.  

The three roles roughly correspond to the three main task categories. You can find more information on the related documentation pages:
* [SYSTEM DEVELOPMENT](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_setup.html)
* [DATA ANALYSIS](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_analysis.html)
* [DATA USE](https://jrc-cbm.readthedocs.io/en/latest/dias4cbm_use.html)