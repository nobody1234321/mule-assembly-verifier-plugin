# Summary

Mule Assembly Verifier is a Maven 3 plugin providing fine-grained validation of the assembly/distribution contents.

## Isn't There a Standard 'Maven' Way of Doing the Same?

Kind of. Recent versions of the assembly plugin added flags for strict filter matching, but it's a drop in the bucket of
other problems like:

* Duplicate entries added to the distribution, resulting in overwrite prompts upon unpacking
* No way to validate where exactly the file landed in the distribution after a complex network of includes/excludes and
  scopes has been applied
* No way to know for sure (and at a glance, for maintainer) which dependency version will be resolved and packaged in the distribution
* No easy overview of the distribution layout
* etc.

This plugin solves above problems, often in a much friendlier way, and on top of that adds the following:

* In case a validation failure is detected, the plugin reports:
  * **Missing** entries from the distribution
  * **Unexpected** entries which we didn't intend to package
  * **Duplicate** entries - whenever a file has been included multiple times in the archive (typically due to the
    _assembly.xml_ configuration errors)
* Usability features
  * Current project's version is inferred and can be used in a validation template (see below)
  * Maven 3-style snapshots are handled transparently (those pesky ones with the datestamp in the name instead of SNAPSHOT)
  * You can skip validation using the skip configuraiton parameter. This allows to disable validation on some profiles/properties.
* Implicit audit of project dependency changes via validation template file commit history (immediately see who upgraded or added/removed
  a jar)


## How Stable is It?

The plugin has been developed for a Mule project and used by its builds for several years now. If it's good enough
for a project with ~100 modules, chances are it's good enough for your case, too :) Recent version had major improvements
and is no longer tied to Mule - the validation became generic. The Mule name is the (beloved) legacy in the name and nothing else.

## Ok, I Saw the Light, "Show Me the Codes"!

### Add plugin to the build

Add a snippet like the one below to your pom's *build/plugins* section (typically the same module where your assembly is created):

```xml
        <plugin>
            <groupId>org.mule.tools</groupId>
            <artifactId>mule-assembly-verifier</artifactId>
            <version>1.4</version>
            <dependencies>
                <!--
                    Declare a compatible groovy dependency for this plugin to avoid
                    conflicts with the main build.
                -->
                <dependency>
                    <groupId>org.codehaus.groovy</groupId>
                    <artifactId>groovy-all</artifactId>
                    <version>1.6.0</version>
                </dependency>
            </dependencies>

            <executions>
                <execution>
                    <phase>verify</phase>
                    <goals>
                        <goal>verify</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
```

The plugin is bound to the **verify** phase of the build, right after the **package**, and before **install**. If the distribution
layout and contents fail to validate, the build will halt and validation report be printed.

### (Optional) Declare a plugin repository

This is only required until a released version is propagated to the maven central (may take a while) or when trying out
a development snapshot:

```xml
    <pluginRepositories>
        <pluginRepository>
            <id>codehaus-plugin-snapshots</id>
            <name>Codehaus Plugin Snapshot Repository</name>
            <url>http://snapshots.repository.codehaus.org</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
            <releases>
                <enabled>false</enabled>
            </releases>
        </pluginRepository>
        <pluginRepository>
            <id>codehaus-plugins</id>
            <name>Codehaus Plugin Repository</name>
            <url>http://repository.codehaus.org</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
            <releases>
                <enabled>true</enabled>
            </releases>
        </pluginRepository>
    </pluginRepositories>
```

### Create a validation template

Put a **assembly-whitelist.txt** file in your module root.

**Tip:** it's easier to start with a blank file and copy/paste the validation failure report for further editing, e.g.:

```
#
# Distribution root
#
/mule-standalone-${productVersion}
/mule-standalone-${productVersion}/apps
/mule-standalone-${productVersion}/bin

#
# /apps
#
/mule-standalone-${productVersion}/apps/default
/mule-standalone-${productVersion}/apps/default/mule-config.xml

/mule-standalone-${productVersion}/lib/opt/commons-lang-2.4.jar
/mule-standalone-${productVersion}/lib/opt/commons-net-2.0.jar
/mule-standalone-${productVersion}/lib/opt/commons-pool-1.5.3.jar

# this is a special 'mandatory wildcard', as we don't really want to
# list every single file generated by a javadoc tool
/mule-enterprise-standalone-${productVersion}/docs/api-enterprise/+
```

#### Basic Syntax

* Paths are relative to the distribution archive root
* Paths are normalized to use forward slash separators - /
* Paths always start with a root - /
* Comments are supported, any line starting with the # symbol is ignored
* Empty lines are ignored
* Current project's artifacts can use **${productVersion}** placeholder. Release and snapshot versions, including
  m3-style timestamped snapshots are supported
* A special **mandatory wildcard** is supported by adding the + sign _at the end of the path_. Typical use case -
  generated content like javadoc where it could change with every build. The plugin will check the content is there
  (and fail if it's missing), but will treat everything under this dir as valid entries

## Configuration Options

Example:

```xml
    <plugin>
        <groupId>org.mule.tools</groupId>
        <artifactId>mule-assembly-verifier</artifactId>
        <version>1.3</version>
        <configuration>
            <projectOutputFile>mule-standalone-nodocs-${project.version}.zip</projectOutputFile>
        </configuration>
    </plugin>
```

Available options:

----------------------------------------------------------------------------------------------------
|**Name**               |**Type**               |**Default**                        |**Description**|
|-----------------------|-----------------------|-----------------------------------|-------------|
|whitelist              |File                   |assembly-whitelist.txt             |Validation template location|
|projectOutputFile      |String                 |${project.build.finalName}.zip     |Archive to validate|
|productVersion         |String                 |${project.version}                 |This project's version|
|maven3StyleSnapshots   |Boolean                |true                               |Disable for Maven 2 builds|
|skip                   |Boolean                |false                              |Disable execution|


## Known Issues

* Maven 2 support is limited and will be removed in the future
* The only supported distribution formats are those that can be processed by unzip. E.g. zip, rar (Resource Archive),
  but not tar.gz
