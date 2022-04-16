# Layout

## Moving Parts

The Nabu Platform has a number of moving parts that make it work.

- Repository: contains everything that will run on the server, this includes reusable Nabu functionality and the specific projects created for clients
- Integrator: a server that connects to a repository and runs the artifacts found within
- Developer: a desktop tool that connects to a server and allows a user to manipulate the writable resources found in the repository. Some resources in the repository (like the nabu distributables) are read-only and can not be manipulated from the developer.

The Nabu Platform is a development platform, it provides features that can be used to build solutions. It is the client project(s) themselves that contain the final solution built for the client. Unless otherwise agreed upon the client, the copyright for the projects themselves already belongs to the client while the copyright of the Nabu Platform is retained by the original owner, the client only gets a license to run it.

## Technologies

### General

The core logic of Nabu is implemented in **Java**. 

We currently run Nabu on java versions ranging from 8 to the latest. Because of ACME, it is advised to use at least 8u101 because lets encrypt certificates were added at that point. Though specific keystores can be configured if needed. There have been known bugs introduced (and later fixed) in java concerning for example javafx (which is the technology that powers the Nabu Developer). Depending on the version you choose, you may or may not run into these bugs. The advise is to always runs the latest fix version of any given major version.

The Integrator and the Developer have a "lib" folder that contains all the core libraries needed to run an empty Nabu installation. One of the design goals in Nabu is to push as much functionality as possible (this includes java code) into the repository in a module system. This allows for a flexible modular composition of the repository depending on the needs of the client. The Developer will stream the repository from the server and does not need direct access to the repository.

A module is packaged as a "nar" file (Nabu ARchive), each archive is itself a maven repository and is signed. Currently the signatures are not enforced, this means a module does not need to be signed with any key, let alone a particular key. If such functionality is added, it will likely be configurable which key(s) are trusted.

The main goal of signing is the ability to detect tampering. If a client tampers with a module, we can no longer provide support for that module.

Some modules contain java code, some modules contain logic built inside nabu itself and some modules might only contain frontend logic. Or any combination thereof.

**Conclusion**: the Integrator and Developer the same for all clients and installations, the repository can contain varying modules which will determine the capabilities that are present.

#### JARs

The layout of the java code repositories that are part of the Nabu Codebase and the dependency management between them are done with **Maven**. The Nabu Integrator itself is a valid maven endpoint which can be used to deploy jars to.

This means all the java code repositories follow the same layout:

- :src/main/java: the java source code
- :src/main/resources: any static resources needed
- :src/test/java: test code (if any)
- :src/test/resources: test resources (if any)

Because each jar file is a maven project, it also contains a pom.xml file. There are hundreds of repositories which requires a centralized version management system when one repository depends on another. This is achieved with a parent pom file that contains all the versions.

Let's take the pom file for the h2 database module as an example:

```xml
<?xml version="1.0"?>
<project
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
	xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<modelVersion>4.0.0</modelVersion>
	<groupId>nabu</groupId>
	<artifactId>eai-module-jdbc-h2</artifactId>
	<version>${nabu-eai-module-jdbc-h2-version}</version>
	<name>eai-module-jdbc-h2</name>

	<parent>
		<groupId>be.nabu</groupId>
		<artifactId>core</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
		</plugins>
	</build>

	<dependencies>
		<!-- The developer dependency is only necessary if you want to edit the 
			artifacts, in production this dependency can be ignored -->
		<dependency>
			<groupId>be.nabu.eai</groupId>
			<artifactId>eai-developer</artifactId>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>be.nabu.eai</groupId>
			<artifactId>eai-repository</artifactId>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>be.nabu.libs.services</groupId>
			<artifactId>services-jdbc</artifactId>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>nabu</groupId>
			<artifactId>eai-module-jdbc-pool</artifactId>
			<scope>provided</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/com.h2database/h2 -->
		<dependency>
		    <groupId>com.h2database</groupId>
		    <artifactId>h2</artifactId>
		    <version>1.4.199</version>
		</dependency>

	</dependencies>

</project>
```

Overall this is a standard POM, but there are a few things to notice:

- the version itself is a variable that is maintained in the parent pom file
- the dependencies to internal libraries do not carry their own version requirement, they instead inherit this from dependency management in the parent pom
- the internal libraries are marked as provided which is crucial, otherwise the nabu server will expect the libraries to be present in the module itself (which they aren't) and will attempt to resolve them

The third party library is added as per usual. Important to note here is that the Nabu Integrator will validate that all the maven dependencies are available in the module folder and will only expose (at the classloader level) the libraries that are listed here. Simply adding a non-declared library to the module will not make it available to the rest of the system.

Some third party libraries are not correctly packaged as a maven library, they are missing the descriptor files that maven-packaged libraries carry in their META-INF folder. In that case, the Nabu Integrator still needs to be able to detect the correct library, it will do this through the naming of the jar. You have to rename the third party library to follow this syntax: groupId-artifactId-version.jar

For example htmlunit: net.sourceforge.htmlunit-htmlunit-core-js-2.27.jar

If the Nabu Integrator can not correctly identify the file, it will try to resolve it from a configured maven endpoint or (if lacking explicit configuration), the default centrally available maven repositories. This will appear in the logs. If it fails to resolve the jar, it will likely not boot correctly, this will also be very evident from the server logs.

**Note**: when the variable versions were originally introduced for dependency management, maven did not support this when generating eclipse projects. It has not been tested since so it is unclear if maven can handle this. For this reason, instead of "mvn eclipse:eclipse" to generate an eclipse project, a custom script is wrapped around this which will make sure the .classpath file is correct. For illustration purposes the script is added here, it (as all scripts relating to nabu) is written in Glue:

```python
string project ?= null
boolean increment ?= true

boolean goUp = true

# the current directory
if (project == null || project == ".")
	project = replace("^.*?([^/]+)$", "", system.pwd())
	system.cd("..")
	goUp = false
else
	# autocomplete may make a dir out of it
	project = replace("[/]+$", "", project)

projects = series()
# do a scan of available projects
for (directory : file.list(directoryRegex: ".*"))
	if (exists(directory + "/pom.xml"))
		pomContent = xml.objectify(read(directory + "/pom.xml"))
		projects = merge(projects, structure(
				directory: directory,
				groupId: pomContent/groupId,
				artifactId: pomContent/artifactId
			))
	
system.cd(project)

# this does not seem to work well with variable versions...
system.mvn("eclipse:eclipse")

string classpath = read(".classpath")

for (project : projects)
	repoPath = "[^>]+/" + replace("\.", "/", project/groupId) + "/" + project/artifactId + "/[^/>]+/" + project/artifactId + "[^>]+"
	if (classpath ~ "(?s).*" + repoPath + ".*")
		echo("Linking project: " + project/artifactId)
		classpath = replace("<classpathentry[^>]*" + repoPath + "[^>]*>", 
			"<classpathentry combineaccessrules='false' kind='src' path='/" + project/directory + "'/>", classpath)
			
write(".classpath", classpath)

if (goUp)
	system.cd("..")
```

#### SPI

Nabu relies heavily on SPI (service provider interface) to automatically pick up implementations. Classloaders can pick up SPI files in modules and will by default expose them to the system. Classloaders can be marked as "encapsulated" at which point they will not expose the internal things like SPI to the wider system. This can be interesting in case there are collissions with third party SPI definitions.

### Developer

The Nabu Developer contains all the same libraries as the integrator and adds a developer library. This developer library is comparable to a browser: it provides a framework that can be used to visualize components. The actual visualization logic resides within the modules.

Developer uses javafx as its display technology. From java 9 onwards this is no longer distributed by default. This means if you run developer with java 8, it will work with a basic script:

```
java -Dversion=2 -Dbe.nabu.eai.repository.cacheModules=true -cp "lib/*:." be.nabu.eai.developer.Main "$@"
```

Once you switch to a higher version, you will need to manually link in the javafx:

```
java -Xmx4096m --module-path /path/to/javafx-sdk-13.0.1/lib --add-modules=ALL-MODULE-PATH -Dshow.hidden=false -Dbe.nabu.eai.repository.cacheModules=true -Dversion=2 -cp "lib/*:." be.nabu.eai.developer.Main "$@"
```

Work is slowly progressing on a redistributable version using jpackage, but this is very low priority currently.

Important parameters to note in the startup parameters:

- :be.nabu.eai.repository.cacheModules: while not mandatory, this can dramatically increase performance of Developer by caching certain things
- :version: this actually pertains to the glue version. Glue currently still allows version 1 unless otherwise specified. This is likely to change in the future so this parameter might not be necessary anymore at some point
- :show.hidden: things like deprecated items can be hidden by default in developer. they will still function, but can not longer be seen. if you toggle this to true, they will be visible.

It is advised to run developer with at least 4gb of RAM.

### Frontend

You can run any frontend technology on top of the Nabu stack, in a time long past we ran an angularjs stack, but all projects in the last 5 years are based on a slightly modified version of VueJS.

Apart from the core VueJS, we also provide a ton of standard components (e.g. form components, popups...), default theming, bootstrapping for mobile, PWA support, routing, swagger clients, json validation,... On top of all this modules we provide Page Builder. This is a visual interface that can be opened in the browser that allows you to build your web application.

Frontend code (javascript, html, css...) _can_ be distributed in modules without appearing in the project files of the client. However, most code (even that distributed in modules) is copied into the client project through "bundles". Bundles are resolved by a glue script that resides in the module "nabu.web.core". This resolving is triggered by the project itself.

A bundle is a descriptor file that indicates which files or other bundles are necessary to create an end result. Each project has one root bundle. Some nabu modules make use of "auto-bundling", these are bundles that are automatically picked up without needing to explicitly define them in the project bundle. Simply adding the correct component to your application triggers this autobundling.

We copy the original files for two reasons:

- to make the project more "stable", apart from autobundles, it requires an explicit action on the developers end to refresh this externally provided code.
- to be able to manipulate the code. The modules that use autobundling are read-only, they can not be modified. If such modifications are not upstreamed however, they will be gone once a refresh is triggered

In a time long past we used "glue css" (gcss) to create styling, it offers features comparable to scss. However, we have long since standardized on the industry standard of scss for our theming requirements.

## Building

The core of the building is based on maven, there are three different types of builds:

- the jar files
- the modules
- the applications (developer & integrator)

To build the jar files you need the "core" parent pom installed, a snapshot of what it currently looks like:

```xml
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>be.nabu</groupId>
	<artifactId>core</artifactId>
	<version>1.0-SNAPSHOT</version>
	<packaging>pom</packaging>

	<properties>
		<!-- General settings -->
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		
		<!-- Versioning -->
		<be.nabu.utils-utils-io-version>1.12-SNAPSHOT</be.nabu.utils-utils-io-version>
		<be.nabu.utils-utils-security-resources-version>1.1</be.nabu.utils-utils-security-resources-version>
		<be.nabu.utils-utils-security-version>1.8-SNAPSHOT</be.nabu.utils-utils-security-version>
		<be.nabu.utils-utils-codec-version>1.6-SNAPSHOT</be.nabu.utils-utils-codec-version>
		<be.nabu.libs.validator-validator-api-version>1.2</be.nabu.libs.validator-validator-api-version>
		<be.nabu.libs.validator-validator-base-version>1.4</be.nabu.libs.validator-validator-base-version>
		<be.nabu.libs.resources-resources-api-version>1.11-SNAPSHOT</be.nabu.libs.resources-resources-api-version>
		<be.nabu.libs.resources-resources-alias-version>1.1</be.nabu.libs.resources-resources-alias-version>
		<be.nabu.libs.resources-resources-cifs-version>1.2</be.nabu.libs.resources-resources-cifs-version>
		<be.nabu.libs.resources-resources-classpath-version>1.1</be.nabu.libs.resources-resources-classpath-version>
		<be.nabu.libs.resources-resources-file-version>1.6-SNAPSHOT</be.nabu.libs.resources-resources-file-version>
		<be.nabu.libs.resources-resources-memory-version>1.2</be.nabu.libs.resources-resources-memory-version>
		<be.nabu.libs.resources-resources-remote-version>1.10-SNAPSHOT</be.nabu.libs.resources-resources-remote-version>
		<be.nabu.libs.resources-resources-url-version>1.1</be.nabu.libs.resources-resources-url-version>
		<be.nabu.libs.resources-resources-zip-version>1.2</be.nabu.libs.resources-resources-zip-version>
		<be.nabu.libs.converter-converter-api-version>1.3</be.nabu.libs.converter-converter-api-version>
		<be.nabu.libs.converter-converter-base-version>1.6-SNAPSHOT</be.nabu.libs.converter-converter-base-version>
		<be.nabu.libs.types-types-api-version>1.8-SNAPSHOT</be.nabu.libs.types-types-api-version>
		<be.nabu.libs.property-property-api-version>1.2-SNAPSHOT</be.nabu.libs.property-property-api-version>
		<be.nabu.libs.artifacts-artifacts-api-version>1.6-SNAPSHOT</be.nabu.libs.artifacts-artifacts-api-version>
		<be.nabu.libs.types-types-base-version>1.18-SNAPSHOT</be.nabu.libs.types-types-base-version>
		<be.nabu.libs.types-types-binding-api-version>1.2</be.nabu.libs.types-types-binding-api-version>
		<be.nabu.libs.types-types-binding-xml-version>1.10-SNAPSHOT</be.nabu.libs.types-types-binding-xml-version>
		<be.nabu.libs.types-types-java-version>1.8-SNAPSHOT</be.nabu.libs.types-types-java-version>
		<be.nabu.libs.types-types-binding-json-version>1.11-SNAPSHOT</be.nabu.libs.types-types-binding-json-version>
		<be.nabu.libs.types-types-binding-form-version>1.0</be.nabu.libs.types-types-binding-form-version>
		<be.nabu.libs.types-types-binding-flat-version>1.5</be.nabu.libs.types-types-binding-flat-version>
		<be.nabu.libs.types-types-definition-api-version>1.0</be.nabu.libs.types-types-definition-api-version>
		<be.nabu.libs.types-types-definition-xml-version>1.6-SNAPSHOT</be.nabu.libs.types-types-definition-xml-version>
		<be.nabu.libs.types-types-structure-version>1.7-SNAPSHOT</be.nabu.libs.types-types-structure-version>
		<be.nabu.libs.types-types-definition-xsd-version>1.4</be.nabu.libs.types-types-definition-xsd-version>
		<be.nabu.libs.events-events-api-version>1.2</be.nabu.libs.events-events-api-version>
		<be.nabu.libs.events-events-impl-version>1.4-SNAPSHOT</be.nabu.libs.events-events-impl-version>
		<be.nabu.libs.wsdl-wsdl-api-version>1.1-SNAPSHOT</be.nabu.libs.wsdl-wsdl-api-version>
		<be.nabu.libs.wsdl-wsdl-formatter-version>1.0</be.nabu.libs.wsdl-wsdl-formatter-version>
		<be.nabu.utils-utils-xml-version>1.1</be.nabu.utils-utils-xml-version>
		<be.nabu.libs.wsdl-wsdl-parser-version>1.1-SNAPSHOT</be.nabu.libs.wsdl-wsdl-parser-version>
		<be.nabu.libs.types-types-xml-version>1.8-SNAPSHOT</be.nabu.libs.types-types-xml-version>
		<be.nabu.libs.evaluator-evaluator-api-version>1.15-SNAPSHOT</be.nabu.libs.evaluator-evaluator-api-version>
		<be.nabu.libs.evaluator-evaluator-types-version>1.7-SNAPSHOT</be.nabu.libs.evaluator-evaluator-types-version>
		<be.nabu.libs.types-types-uml-version>1.8-SNAPSHOT</be.nabu.libs.types-types-uml-version>
		<be.nabu.libs.services-services-api-version>1.13-SNAPSHOT</be.nabu.libs.services-services-api-version>
		<be.nabu.libs.authentication-authentication-api-version>1.11-SNAPSHOT</be.nabu.libs.authentication-authentication-api-version>
		<be.nabu.libs.authentication-authentication-database-version>1.0</be.nabu.libs.authentication-authentication-database-version>
		<be.nabu.libs.authentication-authentication-resources-version>1.0</be.nabu.libs.authentication-authentication-resources-version>
		<be.nabu.libs.http-http-core-version>1.7-SNAPSHOT</be.nabu.libs.http-http-core-version>
		<be.nabu.libs.http-http-api-version>1.10-SNAPSHOT</be.nabu.libs.http-http-api-version>
		<be.nabu.utils-utils-mime-version>1.11-SNAPSHOT</be.nabu.utils-utils-mime-version>
		<be.nabu.libs.nio-nio-server-version>1.18-SNAPSHOT</be.nabu.libs.nio-nio-server-version>
		<be.nabu.libs.metrics-metrics-api-version>1.1-SNAPSHOT</be.nabu.libs.metrics-metrics-api-version>
		<be.nabu.libs.metrics-metrics-core-version>1.5-SNAPSHOT</be.nabu.libs.metrics-metrics-core-version>
		<be.nabu.libs.http-http-server-version>1.19-SNAPSHOT</be.nabu.libs.http-http-server-version>
		<be.nabu.libs.http-http-client-version>1.8-SNAPSHOT</be.nabu.libs.http-http-client-version>
		<be.nabu.libs.cache-cache-api-version>1.7-SNAPSHOT</be.nabu.libs.cache-cache-api-version>
		<be.nabu.libs.cache-cache-resources-version>1.1</be.nabu.libs.cache-cache-resources-version>
		<be.nabu.libs.services-services-jdbc-version>1.14-SNAPSHOT</be.nabu.libs.services-services-jdbc-version>
		<be.nabu.libs.services-services-pojo-version>1.2-SNAPSHOT</be.nabu.libs.services-services-pojo-version>
		<be.nabu.libs.services-services-wsdl-version>1.4-SNAPSHOT</be.nabu.libs.services-services-wsdl-version>
		<be.nabu.libs.services-services-vm-version>1.13-SNAPSHOT</be.nabu.libs.services-services-vm-version>
		<be.nabu.libs.services-services-cache-version>1.1</be.nabu.libs.services-services-cache-version>
		<be.nabu.libs.services-services-mask-version>1.0</be.nabu.libs.services-services-mask-version>
		<be.nabu.libs.types-types-mask-version>1.4-SNAPSHOT</be.nabu.libs.types-types-mask-version>
		<be.nabu.libs.swagger-swagger-api-version>1.4-SNAPSHOT</be.nabu.libs.swagger-swagger-api-version>
		<be.nabu.libs.swagger-swagger-parser-version>1.12-SNAPSHOT</be.nabu.libs.swagger-swagger-parser-version>
		<be.nabu.libs.types-types-map-version>1.4</be.nabu.libs.types-types-map-version>
		<be.nabu.libs.swagger-swagger-formatter-version>1.9-SNAPSHOT</be.nabu.libs.swagger-swagger-formatter-version>
		<be.nabu.utils-utils-aspects-version>1.1</be.nabu.utils-utils-aspects-version>
		<be.nabu.libs.resources-resources-snapshot-version>1.0</be.nabu.libs.resources-resources-snapshot-version>
		<be.nabu.libs.http-http-server-rest-version>1.2</be.nabu.libs.http-http-server-rest-version>
		<be.nabu.libs.http-http-server-websockets-version>1.6-SNAPSHOT</be.nabu.libs.http-http-server-websockets-version>
		<be.nabu.libs.http-http-client-ntlm-version>1.1</be.nabu.libs.http-http-client-ntlm-version>
		<be.nabu.libs.dms-dms-api-version>1.1</be.nabu.libs.dms-dms-api-version>
		<be.nabu.libs.vfs-vfs-api-version>1.0</be.nabu.libs.vfs-vfs-api-version>
		<be.nabu.libs.datastore-datastore-api-version>1.2</be.nabu.libs.datastore-datastore-api-version>
		<be.nabu.libs.dms-dms-core-version>1.5</be.nabu.libs.dms-dms-core-version>
		<be.nabu.libs.vfs-vfs-resources-version>1.0</be.nabu.libs.vfs-vfs-resources-version>
		<be.nabu.glue-glue-api-version>1.10</be.nabu.glue-glue-api-version>
		<be.nabu.glue-glue-version>1.20</be.nabu.glue-glue-version>
		<be.nabu.glue-glue-xml-version>1.0</be.nabu.glue-glue-xml-version>
		<be.nabu.glue-glue-json-version>1.4</be.nabu.glue-glue-json-version>
		<be.nabu.glue-glue-selenese-version>1.3</be.nabu.glue-glue-selenese-version>
		<be.nabu.glue-glue-appium-version>1.0</be.nabu.glue-glue-appium-version>
		<be.nabu.glue-glue-services-version>1.11-SNAPSHOT</be.nabu.glue-glue-services-version>
		<be.nabu.glue-glue-types-version>1.7</be.nabu.glue-glue-types-version>
		<be.nabu.glue-glue-soapui-version>1.0</be.nabu.glue-glue-soapui-version>
		<be.nabu.glue-glue-transcoder-version>1.0</be.nabu.glue-glue-transcoder-version>
		<be.nabu.glue-glue-images-version>1.0</be.nabu.glue-glue-images-version>
		<be.nabu.glue-glue-pdf-version>1.0</be.nabu.glue-glue-pdf-version>
		<be.nabu.glue-glue-console-version>1.0</be.nabu.glue-glue-console-version>
		<be.nabu.glue-glue-validators-version>1.0</be.nabu.glue-glue-validators-version>
		<be.nabu.glue-glue-integrator-version>1.0</be.nabu.glue-glue-integrator-version>
		<be.nabu.glue-glue-events-version>1.1</be.nabu.glue-glue-events-version>
		<be.nabu.glue-glue-database-version>1.0</be.nabu.glue-glue-database-version>
		<be.nabu.glue-glue-excel-version>1.0</be.nabu.glue-glue-excel-version>
		<be.nabu.utils-utils-excel-version>1.3-SNAPSHOT</be.nabu.utils-utils-excel-version>
		<be.nabu.jfx-jfx-control-tree-version>1.6-SNAPSHOT</be.nabu.jfx-jfx-control-tree-version>
		<be.nabu.jfx-jfx-control-ace-version>1.5-SNAPSHOT</be.nabu.jfx-jfx-control-ace-version>
		<be.nabu.libs-html-scraper-version>1.0</be.nabu.libs-html-scraper-version>
		<be.nabu.glue-glue-html-version>1.0</be.nabu.glue-glue-html-version>
		<be.nabu.libs.http-http-glue-version>1.26-SNAPSHOT</be.nabu.libs.http-http-glue-version>
		<be.nabu.jfx-jfx-control-line-version>1.0</be.nabu.jfx-jfx-control-line-version>
		<be.nabu.eai-eai-repository-version>1.22-SNAPSHOT</be.nabu.eai-eai-repository-version>
		<be.nabu.libs.artifacts-artifacts-maven-version>1.3-SNAPSHOT</be.nabu.libs.artifacts-artifacts-maven-version>
		<be.nabu.libs.maven-maven-repository-version>1.4</be.nabu.libs.maven-maven-repository-version>
		<be.nabu.libs.maven-maven-server-version>1.0</be.nabu.libs.maven-maven-server-version>
		<be.nabu.eai-eai-api-version>1.7-SNAPSHOT</be.nabu.eai-eai-api-version>
		<be.nabu.libs.http-http-client-websockets-version>0.2</be.nabu.libs.http-http-client-websockets-version>
		<be.nabu.eai-eai-server-version>1.30-SNAPSHOT</be.nabu.eai-eai-server-version>
		<be.nabu.eai-eai-developer-version>1.19-SNAPSHOT</be.nabu.eai-eai-developer-version>
		<be.nabu.jfx-jfx-control-spinner-version>1.0</be.nabu.jfx-jfx-control-spinner-version>
		<be.nabu.libs.tasks-tasks-api-version>1.2</be.nabu.libs.tasks-tasks-api-version>
		<be.nabu.libs.tasks-tasks-queues-database-version>1.3</be.nabu.libs.tasks-tasks-queues-database-version>
		<be.nabu.libs.tasks-tasks-queues-cache-version>1.0</be.nabu.libs.tasks-tasks-queues-cache-version>
		<be.nabu.libs.tasks-tasks-remote-version>1.1</be.nabu.libs.tasks-tasks-remote-version>
		<be.nabu.libs.datastore-datastore-resources-version>1.3</be.nabu.libs.datastore-datastore-resources-version>
		<be.nabu.libs.channels-channels-api-version>1.6</be.nabu.libs.channels-channels-api-version>
		<be.nabu.libs.datatransactions-datatransactions-api-version>1.2</be.nabu.libs.datatransactions-datatransactions-api-version>
		<be.nabu.libs.channels-channels-resources-version>1.4</be.nabu.libs.channels-channels-resources-version>
		<be.nabu.libs.datatransactions-datatransactions-database-version>1.3</be.nabu.libs.datatransactions-datatransactions-database-version>
		<be.nabu.libs.datastore-datastore-urn-database-version>1.2</be.nabu.libs.datastore-datastore-urn-database-version>
		<be.nabu.libs.http-http-server-kerberos-version>1.0</be.nabu.libs.http-http-server-kerberos-version>
		<nabu-eai-module-auditing-version>1.5-SNAPSHOT</nabu-eai-module-auditing-version>
		<be.nabu.libs.http-http-server-kerberos-scm>https://nablex@bitbucket.org/nablex/http-server-kerberos</be.nabu.libs.http-http-server-kerberos-scm>
		<nabu-eai-module-authentication-kerberos-scm>https://nablex@bitbucket.org/nablex/eai-module-authentication-kerberos</nabu-eai-module-authentication-kerberos-scm>
		<nabu-eai-module-authentication-kerberos-version>1.0</nabu-eai-module-authentication-kerberos-version>
		<nabu-eai-module-web-application-version>1.23-SNAPSHOT</nabu-eai-module-web-application-version>
		<nabu-eai-module-http-server-version>1.15-SNAPSHOT</nabu-eai-module-http-server-version>
		<nabu-eai-module-keystore-version>1.8-SNAPSHOT</nabu-eai-module-keystore-version>
		<nabu-eai-module-authorization-vm-version>1.2-SNAPSHOT</nabu-eai-module-authorization-vm-version>
		<nabu-eai-module-services-interface-version>1.7-SNAPSHOT</nabu-eai-module-services-interface-version>
		<nabu-eai-module-types-structure-version>1.10-SNAPSHOT</nabu-eai-module-types-structure-version>
		<nabu-eai-module-glue-version>1.8-SNAPSHOT</nabu-eai-module-glue-version>
		<be.nabu.glue-glue-series-version>1.0</be.nabu.glue-glue-series-version>
		<nabu-eai-module-web-component-version>1.8-SNAPSHOT</nabu-eai-module-web-component-version>
		<nabu-eai-module-authentication-oauth2-version>1.15-SNAPSHOT</nabu-eai-module-authentication-oauth2-version>
		<nabu-eai-module-http-client-version>1.4-SNAPSHOT</nabu-eai-module-http-client-version>
		<nabu-eai-module-proxy-version>1.0</nabu-eai-module-proxy-version>
		<nabu-eai-module-services-vm-version>1.18-SNAPSHOT</nabu-eai-module-services-vm-version>
		<nabu-eai-module-broker-version>1.8</nabu-eai-module-broker-version>
		<nabu-eai-module-datastore-version>1.6</nabu-eai-module-datastore-version>
		<nabu-eai-module-cache-version>1.5-SNAPSHOT</nabu-eai-module-cache-version>
		<nabu-eai-module-cluster-version>1.10-SNAPSHOT</nabu-eai-module-cluster-version>
		<nabu-eai-module-excel-version>1.6-SNAPSHOT</nabu-eai-module-excel-version>
		<nabu-eai-module-configuration-version>1.4-SNAPSHOT</nabu-eai-module-configuration-version>
		<nabu-eai-module-data-provider-version>1.0</nabu-eai-module-data-provider-version>
		<nabu-eai-module-deployment-version>1.13-SNAPSHOT</nabu-eai-module-deployment-version>
		<nabu-eai-module-jdbc-pool-version>1.10-SNAPSHOT</nabu-eai-module-jdbc-pool-version>
		<nabu-eai-module-maven-version>1.3-SNAPSHOT</nabu-eai-module-maven-version>
		<nabu-eai-module-metrics-version>1.7-SNAPSHOT</nabu-eai-module-metrics-version>
		<be.nabu.libs.metrics-metrics-database-version>1.2</be.nabu.libs.metrics-metrics-database-version>
		<nabu-eai-module-rest-version>1.1</nabu-eai-module-rest-version>
		<nabu-eai-module-rest-client-version>1.10-SNAPSHOT</nabu-eai-module-rest-client-version>
		<nabu-eai-module-rest-provider-version>1.17-SNAPSHOT</nabu-eai-module-rest-provider-version>
		<nabu-eai-module-mime-version>1.4</nabu-eai-module-mime-version>
		<nabu-eai-module-types-xml-version>1.1</nabu-eai-module-types-xml-version>
		<nabu-eai-module-types-uml-version>1.6-SNAPSHOT</nabu-eai-module-types-uml-version>
		<nabu-eai-module-types-simple-version>1.2</nabu-eai-module-types-simple-version>
		<nabu-eai-module-wsdl-client-version>1.2-SNAPSHOT</nabu-eai-module-wsdl-client-version>
		<nabu-eai-module-wsdl-provider-version>1.4-SNAPSHOT</nabu-eai-module-wsdl-provider-version>
		<nabu-eai-module-web-cordova-version>1.2-SNAPSHOT</nabu-eai-module-web-cordova-version>
		<nabu-eai-module-template-text-version>1.4-SNAPSHOT</nabu-eai-module-template-text-version>
		<nabu-eai-module-scheduler-version>1.10</nabu-eai-module-scheduler-version>
		<nabu-eai-module-startup-version>1.4-SNAPSHOT</nabu-eai-module-startup-version>
		<nabu-eai-module-smtp-client-version>1.8</nabu-eai-module-smtp-client-version>
		<nabu-eai-module-swagger-version>1.19-SNAPSHOT</nabu-eai-module-swagger-version>
		<nabu-eai-module-workflow-version>1.20-SNAPSHOT</nabu-eai-module-workflow-version>
		<nabu-eai-module-web-websockets-version>1.6-SNAPSHOT</nabu-eai-module-web-websockets-version>
		<nabu-eai-module-web-resources-version>1.13-SNAPSHOT</nabu-eai-module-web-resources-version>
		<nabu-eai-module-services-jdbc-version>1.11-SNAPSHOT</nabu-eai-module-services-jdbc-version>
		<nabu-eai-module-web-wiki-version>1.6-SNAPSHOT</nabu-eai-module-web-wiki-version>
		<nabu-eai-module-tracer-version>1.6-SNAPSHOT</nabu-eai-module-tracer-version>
		<nabu-eai-module-utils-version>1.17-SNAPSHOT</nabu-eai-module-utils-version>
		<be.nabu.packaging-packaging-integrator-version>1.29-SNAPSHOT</be.nabu.packaging-packaging-integrator-version>
		<be.nabu.packaging-packaging-developer-version>1.20-SNAPSHOT</be.nabu.packaging-packaging-developer-version>
		<nabu-eai-module-glue-testing-version>1.3</nabu-eai-module-glue-testing-version>
		<nabu-eai-module-glue-pdf-viewer-version>1.0</nabu-eai-module-glue-pdf-viewer-version>
		<be.nabu.packaging-packaging-cli-version>1.4-SNAPSHOT</be.nabu.packaging-packaging-cli-version>
		<be.nabu.libs.smtp-smtp-server-version>1.2</be.nabu.libs.smtp-smtp-server-version>
		<nabu-eai-module-channels-version>1.8</nabu-eai-module-channels-version>
		<nabu-eai-module-data-transactions-version>1.1</nabu-eai-module-data-transactions-version>
		<be.nabu.jfx-jfx-control-date-version>1.3-SNAPSHOT</be.nabu.jfx-jfx-control-date-version>
		<nabu-eai-module-binding-flat-version>1.2</nabu-eai-module-binding-flat-version>
		<nabu-eai-module-binding-xml-version>1.3</nabu-eai-module-binding-xml-version>
		<nabu-eai-module-binding-json-version>1.6-SNAPSHOT</nabu-eai-module-binding-json-version>
		<nabu-eai-module-smtp-server-version>1.1</nabu-eai-module-smtp-server-version>
		<be.nabu.libs.smtp-smtp-server-forwarder-version>1.1</be.nabu.libs.smtp-smtp-server-forwarder-version>
		<be.nabu.libs.types-types-binding-csv-version>1.5</be.nabu.libs.types-types-binding-csv-version>
		<nabu-eai-module-binding-csv-version>1.2</nabu-eai-module-binding-csv-version>
		<be.nabu.utils-utils-bully-version>1.1</be.nabu.utils-utils-bully-version>
		<nabu-eai-module-jdbc-oracle-version>1.6-SNAPSHOT</nabu-eai-module-jdbc-oracle-version>
		<nabu-eai-module-jdbc-postgresql-version>1.9-SNAPSHOT</nabu-eai-module-jdbc-postgresql-version>
		<nabu-eai-module-notifier-version>1.5-SNAPSHOT</nabu-eai-module-notifier-version>
		<be.nabu.libs.http-http-jwt-version>1.4-SNAPSHOT</be.nabu.libs.http-http-jwt-version>
		<be.nabu.libs.types-types-view-version>1.0</be.nabu.libs.types-types-view-version>
		<nabu-eai-module-services-jdbc-aggregation-version>1.2</nabu-eai-module-services-jdbc-aggregation-version>
		<nabu-eai-module-glue-console-version>1.1</nabu-eai-module-glue-console-version>
		<be.nabu.libs.http-http-acme-version>1.0</be.nabu.libs.http-http-acme-version>
		<nabu-eai-module-web-analyzer-version>1.0</nabu-eai-module-web-analyzer-version>
		<nabu-eai-module-authentication-remote-version>1.1</nabu-eai-module-authentication-remote-version>
		<nabu-eai-module-web-renderer-version>1.6-SNAPSHOT</nabu-eai-module-web-renderer-version>
		<nabu-eai-module-web-compressor-version>1.2-SNAPSHOT</nabu-eai-module-web-compressor-version>
		<nabu-eai-module-web-sitemap-version>1.1</nabu-eai-module-web-sitemap-version>
		<nabu-eai-module-reports-version>1.0</nabu-eai-module-reports-version>
		<nabu-eai-module-jdbc-mssql-version>1.2</nabu-eai-module-jdbc-mssql-version>
		<nabu-eai-module-executor-version>1.4-SNAPSHOT</nabu-eai-module-executor-version>
		<be.nabu.utils-utils-promise-version>1.0</be.nabu.utils-utils-promise-version>
		<be.nabu.libs.nio-nio-client-version>1.3-SNAPSHOT</be.nabu.libs.nio-nio-client-version>
		<nabu-eai-module-http-reverse-proxy-version>1.3-SNAPSHOT</nabu-eai-module-http-reverse-proxy-version>
		<be.nabu.libs.types-types-binding-excel-version>1.1-SNAPSHOT</be.nabu.libs.types-types-binding-excel-version>
		<be.nabu.libs.cluster-cluster-api-version>1.1-SNAPSHOT</be.nabu.libs.cluster-cluster-api-version>
		<be.nabu.libs.cluster-cluster-hazelcast-version>1.1-SNAPSHOT</be.nabu.libs.cluster-cluster-hazelcast-version>
		<nabu-eai-module-deployment-action-version>1.1</nabu-eai-module-deployment-action-version>
		<be.nabu.libs.http-http-sass-version>1.0</be.nabu.libs.http-http-sass-version>
		<nabu-eai-module-libs-phone-version>1.1-SNAPSHOT</nabu-eai-module-libs-phone-version>
		<be.nabu.libs.resources-resources-sftp-version>1.1</be.nabu.libs.resources-resources-sftp-version>
		<nabu-eai-module-pdf-version>1.0</nabu-eai-module-pdf-version>
		<nabu-eai-module-http-acme-version>1.1</nabu-eai-module-http-acme-version>
		<nabu-eai-module-cms-dynamic-version>1.0</nabu-eai-module-cms-dynamic-version>
		<be.nabu.utils-utils-cep-version>1.1-SNAPSHOT</be.nabu.utils-utils-cep-version>
		<nabu-eai-module-cluster-hazelcast-version>1.1-SNAPSHOT</nabu-eai-module-cluster-hazelcast-version>
		<nabu-eai-module-libs-qr-version>1.0</nabu-eai-module-libs-qr-version>
		<nabu-eai-module-jdbc-h2-version>1.1-SNAPSHOT</nabu-eai-module-jdbc-h2-version>
		<nabu-eai-module-services-scl-version>1.1-SNAPSHOT</nabu-eai-module-services-scl-version>
		<nabu-eai-module-websockets-client-version>1.1-SNAPSHOT</nabu-eai-module-websockets-client-version>
		<nabu-eai-module-waf-version>1.1-SNAPSHOT</nabu-eai-module-waf-version>
		<nabu-eai-module-libs-oshi-version>1.0</nabu-eai-module-libs-oshi-version>
		<nabu-eai-module-services-report-version>1.0-SNAPSHOT</nabu-eai-module-services-report-version>
		<nabu-eai-module-services-crud-version>1.0-SNAPSHOT</nabu-eai-module-services-crud-version>
		<nabu-eai-module-services-profile-version>1.0-SNAPSHOT</nabu-eai-module-services-profile-version>
		<nabu-eai-module-features-version>1.0-SNAPSHOT</nabu-eai-module-features-version>
		<nabu-eai-module-lucene-version>1.0-SNAPSHOT</nabu-eai-module-lucene-version>
		<nabu-eai-developer-data-version>1.0-SNAPSHOT</nabu-eai-developer-data-version>
		<be.nabu.libs.tracer-tracer-api-version>1.0-SNAPSHOT</be.nabu.libs.tracer-tracer-api-version>
		<be.nabu.libs.tracer-tracer-dynatrace-version>1.0-SNAPSHOT</be.nabu.libs.tracer-tracer-dynatrace-version>
		<nabu-eai-module-http-icap-version>1.0-SNAPSHOT</nabu-eai-module-http-icap-version>
		<be.nabu.libs.http-http-icap-version>1.0-SNAPSHOT</be.nabu.libs.http-http-icap-version>
		<nabu-eai-module-data-model-version>1.0-SNAPSHOT</nabu-eai-module-data-model-version>
		<nabu-eai-module-services-insight-version>1.0-SNAPSHOT</nabu-eai-module-services-insight-version>
		<be.nabu.glue-glue-metrics-version>1.0-SNAPSHOT</be.nabu.glue-glue-metrics-version>
		<nabu-eai-module-web-events-version>1.0-SNAPSHOT</nabu-eai-module-web-events-version>
		<be.nabu.triton-triton-version>1.0-SNAPSHOT</be.nabu.triton-triton-version>
		<nabu-eai-module-documentation-version>1.0-SNAPSHOT</nabu-eai-module-documentation-version>
		<nabu-eai-module-jdbc-mysql-version>1.0-SNAPSHOT</nabu-eai-module-jdbc-mysql-version>
		<nabu-eai-module-ftp-client-version>1.0-SNAPSHOT</nabu-eai-module-ftp-client-version>
		<nabu-eai-module-web-renderer2-version>1.0-SNAPSHOT</nabu-eai-module-web-renderer2-version>
	</properties>
	
	<build>
		<plugins>
			<plugin>
				<groupId>org.codehaus.mojo</groupId>
				<artifactId>flatten-maven-plugin</artifactId>
				<configuration>
				</configuration>
				<executions>
					<execution>
						<id>flatten</id>
						<phase>process-resources</phase>
						<goals>
							<goal>flatten</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jar-plugin</artifactId>
				<configuration>
					<archive>
						<manifestEntries>
							<Build-Time>${maven.build.timestamp}</Build-Time>
							<Build-Version>${project.version}</Build-Version>
						</manifestEntries>
					</archive>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-surefire-plugin</artifactId>
				<version>2.17</version>
				<configuration>
					<junitArtifactName>junit:junit</junitArtifactName>
					<encoding>UTF-8</encoding>
					<inputEncoding>UTF-8</inputEncoding>
					<outputEncoding>UTF-8</outputEncoding>
					<argLine>-Dfile.encoding=UTF-8</argLine>
					<useSystemClassLoader>false</useSystemClassLoader>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jarsigner-plugin</artifactId>
				<version>1.4</version>
				<executions>
					<execution>
						<id>sign</id>
						<goals>
							<goal>sign</goal>
							<goal>verify</goal>
						</goals>
					</execution>
				</executions>
				<configuration>
					<keystore>${sign.keystore}</keystore>
					<alias>${sign.alias}</alias>
					<storepass>${sign.storepass}</storepass>
					<keypass>${sign.keypass}</keypass>
					<tsa>${sign.tsa}</tsa>
				</configuration>
			</plugin>
		</plugins>
	</build>
	
	<dependencyManagement>
		<dependencies>
			<!-- External -->
			<dependency>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-api</artifactId>
				<version>1.7.28</version>
			</dependency>
			<dependency>
				<groupId>ch.qos.logback</groupId>
				<artifactId>logback-classic</artifactId>
				<version>1.2.3</version>
				<scope>provided</scope>
			</dependency>
			<dependency>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-simple</artifactId>
				<version>1.7.28</version>
			</dependency>
			<dependency>
				<groupId>javax.validation</groupId>
				<artifactId>validation-api</artifactId>
				<version>1.1.0.Final</version>
			</dependency>
			<dependency>
				<groupId>javax</groupId>
				<artifactId>javaee-api</artifactId>
				<version>6.0</version>
				<scope>provided</scope>
			</dependency>
			<dependency>
				<groupId>javax.ws.rs</groupId>
				<artifactId>javax.ws.rs-api</artifactId>
				<version>2.0.1</version>
			</dependency>
			<dependency>
				<groupId>com.zaxxer</groupId>
				<artifactId>HikariCP</artifactId>
				<version>3.4.1</version>
			</dependency>
			
			<!-- Utils -->
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-io</artifactId>
				<version>${be.nabu.utils-utils-io-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-security-resources</artifactId>
				<version>${be.nabu.utils-utils-security-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-security</artifactId>
				<version>${be.nabu.utils-utils-security-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-codec</artifactId>
				<version>${be.nabu.utils-utils-codec-version}</version>
			</dependency>
			
			<!-- Validator -->
			<dependency>
				<groupId>be.nabu.libs.validator</groupId>
				<artifactId>validator-api</artifactId>
				<version>${be.nabu.libs.validator-validator-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.validator</groupId>
				<artifactId>validator-base</artifactId>
				<version>${be.nabu.libs.validator-validator-base-version}</version>
			</dependency>
			
			<!-- Resources -->
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-api</artifactId>
				<version>${be.nabu.libs.resources-resources-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-alias</artifactId>
				<version>${be.nabu.libs.resources-resources-alias-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-cifs</artifactId>
				<version>${be.nabu.libs.resources-resources-cifs-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-file</artifactId>
				<version>${be.nabu.libs.resources-resources-file-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-classpath</artifactId>
				<version>${be.nabu.libs.resources-resources-classpath-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-memory</artifactId>
				<version>${be.nabu.libs.resources-resources-memory-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-remote</artifactId>
				<version>${be.nabu.libs.resources-resources-remote-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-url</artifactId>
				<version>${be.nabu.libs.resources-resources-url-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-zip</artifactId>
				<version>${be.nabu.libs.resources-resources-zip-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.converter</groupId>
				<artifactId>converter-api</artifactId>
				<version>${be.nabu.libs.converter-converter-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.converter</groupId>
				<artifactId>converter-base</artifactId>
				<version>${be.nabu.libs.converter-converter-base-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-api</artifactId>
				<version>${be.nabu.libs.types-types-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.property</groupId>
				<artifactId>property-api</artifactId>
				<version>${be.nabu.libs.property-property-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.artifacts</groupId>
				<artifactId>artifacts-api</artifactId>
				<version>${be.nabu.libs.artifacts-artifacts-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-base</artifactId>
				<version>${be.nabu.libs.types-types-base-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-api</artifactId>
				<version>${be.nabu.libs.types-types-binding-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-xml</artifactId>
				<version>${be.nabu.libs.types-types-binding-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-java</artifactId>
				<version>${be.nabu.libs.types-types-java-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-json</artifactId>
				<version>${be.nabu.libs.types-types-binding-json-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-form</artifactId>
				<version>${be.nabu.libs.types-types-binding-form-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-flat</artifactId>
				<version>${be.nabu.libs.types-types-binding-flat-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-definition-api</artifactId>
				<version>${be.nabu.libs.types-types-definition-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-definition-xml</artifactId>
				<version>${be.nabu.libs.types-types-definition-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-structure</artifactId>
				<version>${be.nabu.libs.types-types-structure-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-definition-xsd</artifactId>
				<version>${be.nabu.libs.types-types-definition-xsd-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.events</groupId>
				<artifactId>events-api</artifactId>
				<version>${be.nabu.libs.events-events-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.events</groupId>
				<artifactId>events-impl</artifactId>
				<version>${be.nabu.libs.events-events-impl-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.wsdl</groupId>
				<artifactId>wsdl-api</artifactId>
				<version>${be.nabu.libs.wsdl-wsdl-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.wsdl</groupId>
				<artifactId>wsdl-formatter</artifactId>
				<version>${be.nabu.libs.wsdl-wsdl-formatter-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-xml</artifactId>
				<version>${be.nabu.utils-utils-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.wsdl</groupId>
				<artifactId>wsdl-parser</artifactId>
				<version>${be.nabu.libs.wsdl-wsdl-parser-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-xml</artifactId>
				<version>${be.nabu.libs.types-types-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.evaluator</groupId>
				<artifactId>evaluator-api</artifactId>
				<version>${be.nabu.libs.evaluator-evaluator-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.evaluator</groupId>
				<artifactId>evaluator-types</artifactId>
				<version>${be.nabu.libs.evaluator-evaluator-types-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-uml</artifactId>
				<version>${be.nabu.libs.types-types-uml-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-api</artifactId>
				<version>${be.nabu.libs.services-services-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.authentication</groupId>
				<artifactId>authentication-api</artifactId>
				<version>${be.nabu.libs.authentication-authentication-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.authentication</groupId>
				<artifactId>authentication-database</artifactId>
				<version>${be.nabu.libs.authentication-authentication-database-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.authentication</groupId>
				<artifactId>authentication-resources</artifactId>
				<version>${be.nabu.libs.authentication-authentication-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-core</artifactId>
				<version>${be.nabu.libs.http-http-core-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-api</artifactId>
				<version>${be.nabu.libs.http-http-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-mime</artifactId>
				<version>${be.nabu.utils-utils-mime-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.nio</groupId>
				<artifactId>nio-server</artifactId>
				<version>${be.nabu.libs.nio-nio-server-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.metrics</groupId>
				<artifactId>metrics-api</artifactId>
				<version>${be.nabu.libs.metrics-metrics-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.metrics</groupId>
				<artifactId>metrics-core</artifactId>
				<version>${be.nabu.libs.metrics-metrics-core-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-server</artifactId>
				<version>${be.nabu.libs.http-http-server-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-client</artifactId>
				<version>${be.nabu.libs.http-http-client-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.cache</groupId>
				<artifactId>cache-api</artifactId>
				<version>${be.nabu.libs.cache-cache-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.cache</groupId>
				<artifactId>cache-resources</artifactId>
				<version>${be.nabu.libs.cache-cache-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-jdbc</artifactId>
				<version>${be.nabu.libs.services-services-jdbc-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-pojo</artifactId>
				<version>${be.nabu.libs.services-services-pojo-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-wsdl</artifactId>
				<version>${be.nabu.libs.services-services-wsdl-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-vm</artifactId>
				<version>${be.nabu.libs.services-services-vm-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-cache</artifactId>
				<version>${be.nabu.libs.services-services-cache-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.services</groupId>
				<artifactId>services-mask</artifactId>
				<version>${be.nabu.libs.services-services-mask-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-mask</artifactId>
				<version>${be.nabu.libs.types-types-mask-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.swagger</groupId>
				<artifactId>swagger-api</artifactId>
				<version>${be.nabu.libs.swagger-swagger-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.swagger</groupId>
				<artifactId>swagger-parser</artifactId>
				<version>${be.nabu.libs.swagger-swagger-parser-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-map</artifactId>
				<version>${be.nabu.libs.types-types-map-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.swagger</groupId>
				<artifactId>swagger-formatter</artifactId>
				<version>${be.nabu.libs.swagger-swagger-formatter-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-aspects</artifactId>
				<version>${be.nabu.utils-utils-aspects-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-snapshot</artifactId>
				<version>${be.nabu.libs.resources-resources-snapshot-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-server-rest</artifactId>
				<version>${be.nabu.libs.http-http-server-rest-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-server-websockets</artifactId>
				<version>${be.nabu.libs.http-http-server-websockets-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-client-ntlm</artifactId>
				<version>${be.nabu.libs.http-http-client-ntlm-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.dms</groupId>
				<artifactId>dms-api</artifactId>
				<version>${be.nabu.libs.dms-dms-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.vfs</groupId>
				<artifactId>vfs-api</artifactId>
				<version>${be.nabu.libs.vfs-vfs-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.datastore</groupId>
				<artifactId>datastore-api</artifactId>
				<version>${be.nabu.libs.datastore-datastore-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.dms</groupId>
				<artifactId>dms-core</artifactId>
				<version>${be.nabu.libs.dms-dms-core-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.vfs</groupId>
				<artifactId>vfs-resources</artifactId>
				<version>${be.nabu.libs.vfs-vfs-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-api</artifactId>
				<version>${be.nabu.glue-glue-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue</artifactId>
				<version>${be.nabu.glue-glue-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-xml</artifactId>
				<version>${be.nabu.glue-glue-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-json</artifactId>
				<version>${be.nabu.glue-glue-json-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-selenese</artifactId>
				<version>${be.nabu.glue-glue-selenese-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-appium</artifactId>
				<version>${be.nabu.glue-glue-appium-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-services</artifactId>
				<version>${be.nabu.glue-glue-services-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-types</artifactId>
				<version>${be.nabu.glue-glue-types-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-soapui</artifactId>
				<version>${be.nabu.glue-glue-soapui-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-transcoder</artifactId>
				<version>${be.nabu.glue-glue-transcoder-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-images</artifactId>
				<version>${be.nabu.glue-glue-images-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-pdf</artifactId>
				<version>${be.nabu.glue-glue-pdf-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-console</artifactId>
				<version>${be.nabu.glue-glue-console-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-validators</artifactId>
				<version>${be.nabu.glue-glue-validators-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-integrator</artifactId>
				<version>${be.nabu.glue-glue-integrator-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-events</artifactId>
				<version>${be.nabu.glue-glue-events-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-database</artifactId>
				<version>${be.nabu.glue-glue-database-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-excel</artifactId>
				<version>${be.nabu.glue-glue-excel-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-excel</artifactId>
				<version>${be.nabu.utils-utils-excel-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.jfx</groupId>
				<artifactId>jfx-control-tree</artifactId>
				<version>${be.nabu.jfx-jfx-control-tree-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.jfx</groupId>
				<artifactId>jfx-control-ace</artifactId>
				<version>${be.nabu.jfx-jfx-control-ace-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs</groupId>
				<artifactId>html-scraper</artifactId>
				<version>${be.nabu.libs-html-scraper-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-html</artifactId>
				<version>${be.nabu.glue-glue-html-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-glue</artifactId>
				<version>${be.nabu.libs.http-http-glue-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.jfx</groupId>
				<artifactId>jfx-control-line</artifactId>
				<version>${be.nabu.jfx-jfx-control-line-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.eai</groupId>
				<artifactId>eai-repository</artifactId>
				<version>${be.nabu.eai-eai-repository-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.artifacts</groupId>
				<artifactId>artifacts-maven</artifactId>
				<version>${be.nabu.libs.artifacts-artifacts-maven-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.maven</groupId>
				<artifactId>maven-repository</artifactId>
				<version>${be.nabu.libs.maven-maven-repository-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.maven</groupId>
				<artifactId>maven-server</artifactId>
				<version>${be.nabu.libs.maven-maven-server-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.eai</groupId>
				<artifactId>eai-api</artifactId>
				<version>${be.nabu.eai-eai-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-client-websockets</artifactId>
				<version>${be.nabu.libs.http-http-client-websockets-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.eai</groupId>
				<artifactId>eai-server</artifactId>
				<version>${be.nabu.eai-eai-server-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.eai</groupId>
				<artifactId>eai-developer</artifactId>
				<version>${be.nabu.eai-eai-developer-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.jfx</groupId>
				<artifactId>jfx-control-spinner</artifactId>
				<version>${be.nabu.jfx-jfx-control-spinner-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.tasks</groupId>
				<artifactId>tasks-api</artifactId>
				<version>${be.nabu.libs.tasks-tasks-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.tasks</groupId>
				<artifactId>tasks-queues-database</artifactId>
				<version>${be.nabu.libs.tasks-tasks-queues-database-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.tasks</groupId>
				<artifactId>tasks-queues-cache</artifactId>
				<version>${be.nabu.libs.tasks-tasks-queues-cache-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.tasks</groupId>
				<artifactId>tasks-remote</artifactId>
				<version>${be.nabu.libs.tasks-tasks-remote-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.datastore</groupId>
				<artifactId>datastore-resources</artifactId>
				<version>${be.nabu.libs.datastore-datastore-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.channels</groupId>
				<artifactId>channels-api</artifactId>
				<version>${be.nabu.libs.channels-channels-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.datatransactions</groupId>
				<artifactId>datatransactions-api</artifactId>
				<version>${be.nabu.libs.datatransactions-datatransactions-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.channels</groupId>
				<artifactId>channels-resources</artifactId>
				<version>${be.nabu.libs.channels-channels-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.datatransactions</groupId>
				<artifactId>datatransactions-database</artifactId>
				<version>${be.nabu.libs.datatransactions-datatransactions-database-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.datastore</groupId>
				<artifactId>datastore-urn-database</artifactId>
				<version>${be.nabu.libs.datastore-datastore-urn-database-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-server-kerberos</artifactId>
				<version>${be.nabu.libs.http-http-server-kerberos-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-auditing</artifactId>
				<version>${nabu-eai-module-auditing-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-authentication-kerberos</artifactId>
				<version>${nabu-eai-module-authentication-kerberos-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-application</artifactId>
				<version>${nabu-eai-module-web-application-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-http-server</artifactId>
				<version>${nabu-eai-module-http-server-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-keystore</artifactId>
				<version>${nabu-eai-module-keystore-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-authorization-vm</artifactId>
				<version>${nabu-eai-module-authorization-vm-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-interface</artifactId>
				<version>${nabu-eai-module-services-interface-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-types-structure</artifactId>
				<version>${nabu-eai-module-types-structure-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-glue</artifactId>
				<version>${nabu-eai-module-glue-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-series</artifactId>
				<version>${be.nabu.glue-glue-series-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-component</artifactId>
				<version>${nabu-eai-module-web-component-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-authentication-oauth2</artifactId>
				<version>${nabu-eai-module-authentication-oauth2-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-http-client</artifactId>
				<version>${nabu-eai-module-http-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-proxy</artifactId>
				<version>${nabu-eai-module-proxy-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-vm</artifactId>
				<version>${nabu-eai-module-services-vm-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-broker</artifactId>
				<version>${nabu-eai-module-broker-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-datastore</artifactId>
				<version>${nabu-eai-module-datastore-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-cache</artifactId>
				<version>${nabu-eai-module-cache-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-cluster</artifactId>
				<version>${nabu-eai-module-cluster-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-excel</artifactId>
				<version>${nabu-eai-module-excel-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-configuration</artifactId>
				<version>${nabu-eai-module-configuration-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-data-provider</artifactId>
				<version>${nabu-eai-module-data-provider-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-deployment</artifactId>
				<version>${nabu-eai-module-deployment-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-jdbc-pool</artifactId>
				<version>${nabu-eai-module-jdbc-pool-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-maven</artifactId>
				<version>${nabu-eai-module-maven-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-metrics</artifactId>
				<version>${nabu-eai-module-metrics-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.metrics</groupId>
				<artifactId>metrics-database</artifactId>
				<version>${be.nabu.libs.metrics-metrics-database-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-rest</artifactId>
				<version>${nabu-eai-module-rest-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-rest-client</artifactId>
				<version>${nabu-eai-module-rest-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-rest-provider</artifactId>
				<version>${nabu-eai-module-rest-provider-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-mime</artifactId>
				<version>${nabu-eai-module-mime-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-types-xml</artifactId>
				<version>${nabu-eai-module-types-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-types-uml</artifactId>
				<version>${nabu-eai-module-types-uml-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-types-simple</artifactId>
				<version>${nabu-eai-module-types-simple-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-wsdl-client</artifactId>
				<version>${nabu-eai-module-wsdl-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-wsdl-provider</artifactId>
				<version>${nabu-eai-module-wsdl-provider-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-cordova</artifactId>
				<version>${nabu-eai-module-web-cordova-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-template-text</artifactId>
				<version>${nabu-eai-module-template-text-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-scheduler</artifactId>
				<version>${nabu-eai-module-scheduler-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-startup</artifactId>
				<version>${nabu-eai-module-startup-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-smtp-client</artifactId>
				<version>${nabu-eai-module-smtp-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-swagger</artifactId>
				<version>${nabu-eai-module-swagger-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-workflow</artifactId>
				<version>${nabu-eai-module-workflow-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-websockets</artifactId>
				<version>${nabu-eai-module-web-websockets-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-resources</artifactId>
				<version>${nabu-eai-module-web-resources-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-jdbc</artifactId>
				<version>${nabu-eai-module-services-jdbc-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-wiki</artifactId>
				<version>${nabu-eai-module-web-wiki-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-tracer</artifactId>
				<version>${nabu-eai-module-tracer-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-utils</artifactId>
				<version>${nabu-eai-module-utils-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.packaging</groupId>
				<artifactId>packaging-integrator</artifactId>
				<version>${be.nabu.packaging-packaging-integrator-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.packaging</groupId>
				<artifactId>packaging-developer</artifactId>
				<version>${be.nabu.packaging-packaging-developer-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-glue-testing</artifactId>
				<version>${nabu-eai-module-glue-testing-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-glue-pdf-viewer</artifactId>
				<version>${nabu-eai-module-glue-pdf-viewer-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.packaging</groupId>
				<artifactId>packaging-cli</artifactId>
				<version>${be.nabu.packaging-packaging-cli-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.smtp</groupId>
				<artifactId>smtp-server</artifactId>
				<version>${be.nabu.libs.smtp-smtp-server-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-channels</artifactId>
				<version>${nabu-eai-module-channels-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-data-transactions</artifactId>
				<version>${nabu-eai-module-data-transactions-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.jfx</groupId>
				<artifactId>jfx-control-date</artifactId>
				<version>${be.nabu.jfx-jfx-control-date-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-binding-flat</artifactId>
				<version>${nabu-eai-module-binding-flat-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-binding-xml</artifactId>
				<version>${nabu-eai-module-binding-xml-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-binding-json</artifactId>
				<version>${nabu-eai-module-binding-json-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-smtp-server</artifactId>
				<version>${nabu-eai-module-smtp-server-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.smtp</groupId>
				<artifactId>smtp-server-forwarder</artifactId>
				<version>${be.nabu.libs.smtp-smtp-server-forwarder-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-csv</artifactId>
				<version>${be.nabu.libs.types-types-binding-csv-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-binding-csv</artifactId>
				<version>${nabu-eai-module-binding-csv-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-bully</artifactId>
				<version>${be.nabu.utils-utils-bully-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-jdbc-oracle</artifactId>
				<version>${nabu-eai-module-jdbc-oracle-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-jdbc-postgresql</artifactId>
				<version>${nabu-eai-module-jdbc-postgresql-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-notifier</artifactId>
				<version>${nabu-eai-module-notifier-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-jwt</artifactId>
				<version>${be.nabu.libs.http-http-jwt-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-view</artifactId>
				<version>${be.nabu.libs.types-types-view-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-jdbc-aggregation</artifactId>
				<version>${nabu-eai-module-services-jdbc-aggregation-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-glue-console</artifactId>
				<version>${nabu-eai-module-glue-console-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-acme</artifactId>
				<version>${be.nabu.libs.http-http-acme-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-analyzer</artifactId>
				<version>${nabu-eai-module-web-analyzer-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-authentication-remote</artifactId>
				<version>${nabu-eai-module-authentication-remote-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-renderer</artifactId>
				<version>${nabu-eai-module-web-renderer-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-compressor</artifactId>
				<version>${nabu-eai-module-web-compressor-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-sitemap</artifactId>
				<version>${nabu-eai-module-web-sitemap-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-reports</artifactId>
				<version>${nabu-eai-module-reports-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-jdbc-mssql</artifactId>
				<version>${nabu-eai-module-jdbc-mssql-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-executor</artifactId>
				<version>${nabu-eai-module-executor-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-promise</artifactId>
				<version>${be.nabu.utils-utils-promise-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.nio</groupId>
				<artifactId>nio-client</artifactId>
				<version>${be.nabu.libs.nio-nio-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-http-reverse-proxy</artifactId>
				<version>${nabu-eai-module-http-reverse-proxy-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.types</groupId>
				<artifactId>types-binding-excel</artifactId>
				<version>${be.nabu.libs.types-types-binding-excel-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.cluster</groupId>
				<artifactId>cluster-api</artifactId>
				<version>${be.nabu.libs.cluster-cluster-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.cluster</groupId>
				<artifactId>cluster-hazelcast</artifactId>
				<version>${be.nabu.libs.cluster-cluster-hazelcast-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-deployment-action</artifactId>
				<version>${nabu-eai-module-deployment-action-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-sass</artifactId>
				<version>${be.nabu.libs.http-http-sass-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-libs-phone</artifactId>
				<version>${nabu-eai-module-libs-phone-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.resources</groupId>
				<artifactId>resources-sftp</artifactId>
				<version>${be.nabu.libs.resources-resources-sftp-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-pdf</artifactId>
				<version>${nabu-eai-module-pdf-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-http-acme</artifactId>
				<version>${nabu-eai-module-http-acme-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-cms-dynamic</artifactId>
				<version>${nabu-eai-module-cms-dynamic-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.utils</groupId>
				<artifactId>utils-cep</artifactId>
				<version>${be.nabu.utils-utils-cep-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-cluster-hazelcast</artifactId>
				<version>${nabu-eai-module-cluster-hazelcast-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-libs-qr</artifactId>
				<version>${nabu-eai-module-libs-qr-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-jdbc-h2</artifactId>
				<version>${nabu-eai-module-jdbc-h2-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-scl</artifactId>
				<version>${nabu-eai-module-services-scl-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-websockets-client</artifactId>
				<version>${nabu-eai-module-websockets-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-waf</artifactId>
				<version>${nabu-eai-module-waf-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-libs-oshi</artifactId>
				<version>${nabu-eai-module-libs-oshi-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-report</artifactId>
				<version>${nabu-eai-module-services-report-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-crud</artifactId>
				<version>${nabu-eai-module-services-crud-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-profile</artifactId>
				<version>${nabu-eai-module-services-profile-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-features</artifactId>
				<version>${nabu-eai-module-features-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-lucene</artifactId>
				<version>${nabu-eai-module-lucene-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-developer-data</artifactId>
				<version>${nabu-eai-developer-data-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.tracer</groupId>
				<artifactId>tracer-api</artifactId>
				<version>${be.nabu.libs.tracer-tracer-api-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.tracer</groupId>
				<artifactId>tracer-dynatrace</artifactId>
				<version>${be.nabu.libs.tracer-tracer-dynatrace-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-http-icap</artifactId>
				<version>${nabu-eai-module-http-icap-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.libs.http</groupId>
				<artifactId>http-icap</artifactId>
				<version>${be.nabu.libs.http-http-icap-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-data-model</artifactId>
				<version>${nabu-eai-module-data-model-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-services-insight</artifactId>
				<version>${nabu-eai-module-services-insight-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.glue</groupId>
				<artifactId>glue-metrics</artifactId>
				<version>${be.nabu.glue-glue-metrics-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-events</artifactId>
				<version>${nabu-eai-module-web-events-version}</version>
			</dependency>
			<dependency>
				<groupId>be.nabu.triton</groupId>
				<artifactId>triton</artifactId>
				<version>${be.nabu.triton-triton-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-documentation</artifactId>
				<version>${nabu-eai-module-documentation-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-jdbc-mysql</artifactId>
				<version>${nabu-eai-module-jdbc-mysql-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-ftp-client</artifactId>
				<version>${nabu-eai-module-ftp-client-version}</version>
			</dependency>
			<dependency>
				<groupId>nabu</groupId>
				<artifactId>eai-module-web-renderer2</artifactId>
				<version>${nabu-eai-module-web-renderer2-version}</version>
			</dependency>
		</dependencies>
	</dependencyManagement>
	
	<dependencies>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.11</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
	
	<distributionManagement>
		<repository>
			<uniqueVersion>false</uniqueVersion>
			<id>nabu</id>
			<name>Nabu Maven Repository</name>
			<url>http://localhost:5555/maven</url>
			<layout>default</layout>
		</repository>
	</distributionManagement>
</project>
```

Performing a classic "mvn package" on a nabu java project should result in a correctly packaged module. If you have configured the distribution management to point to a valid nabu server, you can also perform a "mvn deploy" though it does require a correctly configured "Maven Library" module, otherwise the nabu server does not know where to put the file.

Glue scripts are used to manipulate the versions inside the core pom file.

For the modules, another parent pom is used but the same principles apply. You can also run a "mvn package" on these packages to create a jar file. Simply renaming this to ".nar" should result in a correct nar file.

To build the applications we use maven assembly. For example the packaging of the integrator:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
	xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<modelVersion>4.0.0</modelVersion>
	<groupId>be.nabu.packaging</groupId>
	<artifactId>packaging-integrator</artifactId>
	<version>${be.nabu.packaging-packaging-integrator-version}</version>
	<name>Nabu Integrator</name>
	<url>http://maven.apache.org</url>
	
	<parent>
		<groupId>be.nabu</groupId>
		<artifactId>core</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-dependency-plugin</artifactId>
				<executions>
					<execution>
						<id>copy-dependencies</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
						<configuration>
							<outputDirectory>${project.build.directory}/lib</outputDirectory>
							<overWriteReleases>true</overWriteReleases>
							<overWriteSnapshots>true</overWriteSnapshots>
							<overWriteIfNewer>true</overWriteIfNewer>
						</configuration>
					</execution>
				</executions>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-assembly-plugin</artifactId>
				<executions>
					<execution>
						<id>assemblyZips</id>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
						<configuration>
							<appendAssemblyId>false</appendAssemblyId>
							<finalName>integrator-${project.version}</finalName>
							<descriptors>
								<descriptor>src/main/assembly/nabu-integrator-assembly.xml</descriptor>
							</descriptors>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
	
	<dependencies>
		<dependency>
			<groupId>be.nabu.eai</groupId>
			<artifactId>eai-server</artifactId>
		</dependency>
		<dependency>
			<groupId>ch.qos.logback</groupId>
			<artifactId>logback-classic</artifactId>
			<scope>compile</scope>
		</dependency>
	</dependencies>
</project>
```

The developer packaging follows a similar pattern:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
	xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<modelVersion>4.0.0</modelVersion>
	<groupId>be.nabu.packaging</groupId>
	<artifactId>packaging-developer</artifactId>
	<version>${be.nabu.packaging-packaging-developer-version}</version>
	<name>Nabu Developer</name>
	<url>http://maven.apache.org</url>

	<parent>
		<groupId>be.nabu</groupId>
		<artifactId>core</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-dependency-plugin</artifactId>
				<executions>
					<execution>
						<id>copy-dependencies</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
						<configuration>
							<outputDirectory>${project.build.directory}/lib</outputDirectory>
							<overWriteReleases>true</overWriteReleases>
							<overWriteSnapshots>true</overWriteSnapshots>
							<overWriteIfNewer>true</overWriteIfNewer>
						</configuration>
					</execution>
				</executions>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-assembly-plugin</artifactId>
				<executions>
					<execution>
						<id>assemblyZips</id>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
						<configuration>
							<appendAssemblyId>false</appendAssemblyId>
							<finalName>developer-${project.version}</finalName>
							<descriptors>
								<descriptor>src/main/assembly/nabu-developer-assembly.xml</descriptor>
							</descriptors>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
	<dependencies>
		<dependency>
			<groupId>be.nabu.eai</groupId>
			<artifactId>eai-developer</artifactId>
		</dependency>
		<dependency>
			<groupId>ch.qos.logback</groupId>
			<artifactId>logback-classic</artifactId>
			<scope>compile</scope>
		</dependency>
	</dependencies>
</project>
```

Executing "mvn package" should also result in a correctly built application.
