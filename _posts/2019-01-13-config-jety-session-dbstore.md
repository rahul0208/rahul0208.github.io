---
layout: post
title:  Configure Database Store for Jetty sessions.
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
#tags: [test]
---

I have a `kata` application, which allows me to be hands-on with various things. This is a web application and sometime back it had the use case of  *Persisting sessions*. The application is a simple webapp running on Jetty,  which is configured via the `jetty-maven-plugin`. Jetty has good support of session persistance. If you look into the docs, it allows you to perssist session in files, db, mongodb, hazelcast  etc. So I went ahead and configured [File Persistance](https://www.eclipse.org/jetty/documentation/9.4.x/configuring-sessions-file-system.html).

Jetty has its own DI sytax which is not intutive enough. so to get `FilePersistance` working I had take [hints](https://www.eclipse.org/jetty/documentation/9.4.x/sessions-usecases.html) to arrive at the following configuration :
```
<Configure id="webapp" class="org.eclipse.jetty.webapp.WebAppContext">
    <Get id="sh" name="sessionHandler">
        <Set name="sessionCache">
            <New class="org.eclipse.jetty.server.session.DefaultSessionCache">
                <Arg><Ref id="sh"/></Arg>
                <Set name="sessionDataStore">
                    <New class="org.eclipse.jetty.server.session.FileSessionDataStore">
                        <Set name="storeDir">
                            <New class="java.io.File">
                                <Arg>/tmp/jetty-sessions</Arg>
                            </New>
                        </Set>
                    </New>
                </Set>
            </New>
        </Set>
    </Get>
</Configure
```

The above configuration  can be added to `jetty-web.xml` in classpath or the `jetty-maven-plugin` can be configured to pick the specific configuration file in the following manner.

```
<plugin>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-maven-plugin</artifactId>
    <version>9.4.14.v20181114</version>
    <configuration>
        <contextXml>${project.build.directory}/classes/jetty.xml</contextXml>
    </configuration>
</plugin>
```

This was fine untill I need to change the store to my Database as I was running 2 nodes and needed to make sure that session information is available. I spent a good amount of time looking at the dtd to get it working. I had to accomplish the follwoing two steps :

- Configure a JNDI datasource for my database
- Configure JDBCSessionDataStore to write using the created JNDI Resource

The major task was to understand how do I need to configure JNDI first. I had to spend some time to understand  the difference between `Call` and `Set`. Also it wasn't very clear that the xml can have ONLY one `Configure` element. Anyways after spending a substantial amout of time I got the following working configuration it place.

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">


<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <Set name="contextPath">/</Set>
    <!--Set name="war">/home/home/Projects/assessmatic/admin/target/admin.war</Set-->

    <Call name="addBean">
        <Arg>
            <New id="postgres" class="org.eclipse.jetty.plus.jndi.Resource">
                <Arg></Arg>
                <Arg>/jdbc/sessions</Arg>
                <Arg>
                    <New class="org.postgresql.ds.PGSimpleDataSource">
                        <Set name="User">postgres</Set>
                        <Set name="Password">password</Set>
                        <Set name="DatabaseName">postgres</Set>
                        <Set name="ServerName">172.17.0.2</Set>
                        <Set name="PortNumber">5432</Set>
                    </New>
                </Arg>
            </New>
        </Arg>
    </Call>

    <Call id="sh" name="getSessionHandler">
        <Set name="sessionCache">
            <New class="org.eclipse.jetty.server.session.DefaultSessionCache">
                <Arg>
                    <Ref id="sh"/>
                </Arg>
                <Set name="sessionDataStore">
                    <New class="org.eclipse.jetty.server.session.JDBCSessionDataStore">
                        <Set name="gracePeriodSec">
                            <Property name="jetty.session.gracePeriod.seconds" default="3600"/>
                        </Set>
                        <Set name="savePeriodSec">
                            <Property name="jetty.session.savePeriod.seconds" default="0"/>
                        </Set>
                        <Set name="databaseAdaptor">
                            <New class="org.eclipse.jetty.server.session.DatabaseAdaptor">
                                <Set name="DatasourceName">
                                    <Property name="jetty.session.jdbc.datasourceName" default="/jdbc/sessions"/>
                                </Set>
                                <Set name="blobType">
                                    <Property name="jetty.session.jdbc.blobType"/>
                                </Set>
                                <Set name="longType">
                                    <Property name="jetty.session.jdbc.longType"/>
                                </Set>
                                <Set name="stringType">
                                    <Property name="jetty.session.jdbc.stringType"/>
                                </Set>
                            </New>
                        </Set>
                        <Set name="sessionTableSchema">
                            <New
                                    class="org.eclipse.jetty.server.session.JDBCSessionDataStore$SessionTableSchema">
                                <Set name="accessTimeColumn">
                                    <Property name="jetty.session.jdbc.schema.accessTimeColumn" default="accessTime"/>
                                </Set>
                                <Set name="contextPathColumn">
                                    <Property name="jetty.session.jdbc.schema.contextPathColumn" default="contextPath"/>
                                </Set>
                                <Set name="cookieTimeColumn">
                                    <Property name="jetty.session.jdbc.schema.cookieTimeColumn" default="cookieTime"/>
                                </Set>
                                <Set name="createTimeColumn">
                                    <Property name="jetty.session.jdbc.schema.createTimeColumn" default="createTime"/>
                                </Set>
                                <Set name="expiryTimeColumn">
                                    <Property name="jetty.session.jdbc.schema.expiryTimeColumn" default="expiryTime"/>
                                </Set>
                                <Set name="lastAccessTimeColumn">
                                    <Property name="jetty.session.jdbc.schema.lastAccessTimeColumn"
                                              default="lastAccessTime"/>
                                </Set>
                                <Set name="lastSavedTimeColumn">
                                    <Property name="jetty.session.jdbc.schema.lastSavedTimeColumn"
                                              default="lastSavedTime"/>
                                </Set>
                                <Set name="idColumn">
                                    <Property name="jetty.session.jdbc.schema.idColumn" default="sessionId"/>
                                </Set>
                                <Set name="lastNodeColumn">
                                    <Property name="jetty.session.jdbc.schema.lastNodeColumn" default="lastNode"/>
                                </Set>
                                <Set name="virtualHostColumn">
                                    <Property name="jetty.session.jdbc.schema.virtualHostColumn" default="virtualHost"/>
                                </Set>
                                <Set name="maxIntervalColumn">
                                    <Property name="jetty.session.jdbc.schema.maxIntervalColumn" default="maxInterval"/>
                                </Set>
                                <Set name="mapColumn">
                                    <Property name="jetty.session.jdbc.schema.mapColumn" default="map"/>
                                </Set>
                                <Set name="tableName">
                                    <Property name="jetty.session.jdbc.schema.table" default="JettySessions"/>
                                </Set>
                            </New>
                        </Set>
                    </New>
                </Set>
            </New>
        </Set>
    </Call>
</Configure>
```
Now this had a few classpath issues as the plugin is now first creating JNDI resource before loading the application. So I had to add `postgres` dependencies to its dependency configuration. Once configured the session persistance works very well.
