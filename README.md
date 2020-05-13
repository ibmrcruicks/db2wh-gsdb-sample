# db2wh-gsdb-sample
steps to load GO SALES sample db to DB2 Warehouse on Cloud from MACOS

For maby years, IBM has provided a rich and realistic database to help users become familiar with database applications and data warehousing - the Great Outdoors Sales database, or GSDB.

Current deployments of Db2 Warehouse on Cloud (db2wh) do not include any sample data; you can still obtain the "GO SALES" sample from [IBM Software downloads - GSDB](ftp://ftp.software.ibm.com/software/data/sample/GSDB_DB2_LUW_ZOS_v2r3.zip), and apply that manually to db2wh.

In the past, this would have been relatively straightforward on MACOS, but IBM no longer provides DB2 Server or Connect products on MACOS; an alternative strategy is needed to use a MACOS client to setup the GSDB sample database.

two main options exist -
+ install the Data Server driver package, with CLPPlus and the db2cli utilities
+ create a docker instance of Db2, get access to all the tools and utilities, and get a local database server to play with

The first option ultimately proved unhelpful as it does not provide a native db2 command processor (needed by the GSDB install scripts). The driver package is intended for user-written applications and 3rd-party utilities (e.g. Excel) that make use of ODBC and JDBC.

The second option is much more appealling - a [self-contained Db2 instance](https://hub.docker.com/r/ibmcom/db2) running in Linux, with a full set of command-line tools and database utilities. Installation and activiation instructions are clear and simple:
```
docker pull ibmcom/db2:latest
docker run -itd --name mydb2 --privileged=true -p 50000:50000 -e LICENSE=accept -e DB2INST1_PASSWORD=<choose an instance password> -e DBNAME=testdb -v <db storage dir>:/database ibmcom/db2
```
Once the DB2INST1 instance has finished installing, you will be able to use the db2 command processor to create a local database alias for the Db2 Warehouse on Cloud instance:
+ `docker exec -ti mydb2 bash -c "su - ${DB2INSTANCE}"`
+ `db2 catalog tcpip node DB2WH remote <instance credentials: hostname> service 50001 security SSL`
+ `db2 catalog database BLUDB as BLUDB at node DB2WH authentication server`

At this point, if you try to connect to the database , you will likely receive a [SQL30081N error](https://www.ibm.com/support/pages/sql30081n-tcpip-communication-errors), probably with SQLSTATE=08001; the explanation does not really identify the problem -- lack of SSL support in the default db2 command processor.

Follow instructions for [enabling SSL connectivity](https://cloud.ibm.com/docs/services/Db2whc?topic=Db2whc-ssl_support) and your connection will (almost) work. Although the GSkit does not appear to be installed in the container image, the required utilities are provided by Db2 in the `sqllib/gskit/bin/` directory.

A `db2 connect` attempt at this poinb will fail with a `GSKit Error: 201` message - again, the explanation doesn't identify the problem -- missing information about the password stash for the keystore. To fix this, issue an additional command, similar to the following, to set the database configuration variable:
`db2 update dbm cfg using SSL_CLNT_STASH c:\PROGRA~1\IBM\gsk8\mykeystore.sth`
(adjust for the name/location of your keystore stash file, as you did for SSL_CLNT_KEYDB).

Now you should be able to issue a connection request:
`db2 connect to BLUDB user <instance credentials: username> using <instance credentials: password>`
and get a confirmation response like:
```
   Database Connection Information

 Database server        = DB2/LINUXX8664 11.1.9.0
 SQL authorization ID   = BLUADMIN
 Local database alias   = BLUDB
```

If all is well at this point, installing the GSDB sample database should be possible.

Download the [GSDB zip](ftp://ftp.software.ibm.com/software/data/sample/GSDB_DB2_LUW_ZOS_v2r3.zip) into the docker container, using curl or wget, or copy from your MACOS host Download directory into the <db storage dir> specified when initializing the Db2 instance.
  
Unzip the GSDB package, and make a couple of adjustments to the install scripts in the *unix* directory.
+ edit *GS_SCRIPTS/RUN_GOSALES_IMPORT.sh* to make sure *GOSALES_XML_SUPPORT* is always *N* - db2wh does not appear to have been configured with XML datatype support
+ edit *GOSalesConfig.sh* to set *GOSALES_DPF* to *Y* - this avoids LOAD warnings when populating the CUST_CUSTOMER table with large VARCHAR data.

If GOSALES_DPF defaults to N, the load process will execute until all tables are loaded (except CUST_CUSTOMER), and all setup for contraints and views will be bypassed. 

You will find schema information for GSDB in the documentation for the current/last version of [IBM Data Studio](https://www.ibm.com/support/knowledgecenter/SS62YD_4.1.1/com.ibm.sampledata.go.doc/topics/greatoutdoors.html)



