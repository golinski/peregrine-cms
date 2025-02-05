# Apache Sling Launchpad Builder for Peregrine CMS

The Launchpad Builder project produces both a Standalone Java Application which
contains everything needed to run the Launchpad in a single JAR file and a Web
Application for Peregrine CMS. It does only provide necessary OSGi bundles
and the Peregrine Sling Packages must be installed later.

## How to run the Sling launchpad/builder module in Standalone mode
-------------------------------------------------------------------

  NOTE: "mvn clean" deletes the "sling" work directory in the project base
        directory. It is advisable to use a work directory outside of the
        project directory.

1) Build Sling using 

	mvn clean install
	
in the top-level directory of the Sling source code.

2) Start the generated jar with

	 java -jar target/com.peregrine-cms.sling.launchpad-9.1.jar 
	 
Use the correct version number instead of 9.1, if needed.

3) Browse Sling in:

        http://localhost:8080

## How to run the Sling launchpad/builder module in webapp mode
---------------------------------------------------------------

1) Build Sling using 

	mvn clean install
	
in the top-level directory of the Sling source code.

2) Start the generated war with

	 mvn jetty:run-war

3) Browse Sling in:

        http://localhost:8888
        
  OR
  
   Deploy target/com.peregrine-cms.sling.launchpad-9.1.jar to your favorite application
   server or servlet container.
