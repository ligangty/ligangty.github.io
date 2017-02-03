---
layout: post
title: Enable gzip http(s) compression in different container
---

{{ page.title }}
================

* Jboss AS 7+  (EAP 6+):

  in ${JBoss_Home}/standalone/configuration/standalone.xml, set following:

      <system-properties>
      .........
      <property name="org.apache.coyote.http11.Http11Protocol.COMPRESSION" value="on"/>
      <property name="org.apache.coyote.http11.Http11Protocol.COMPRESSION_MIME_TYPES" value="text/plain,text/javascript,text/css,text/html,text/xml,text/json,application/javascript,application/json,application/xml"/>
      .........
      </system-properties>


* JBoss AS 6- (EAP 5-)

  in  ${JBoss_Home}/server/default/deploy/jboss-web.deployer/server.xml, add following:

      <Connector port="8080"
      address="${jboss.bind.address}" maxThreads="250"
      maxHttpHeaderSize="8192" emptySessionPath="true"
      protocol="HTTP/1.1" enableLookups="false" redirectPort="8443" acceptCount="100"  
      connectionTimeout="20000" disableUploadTimeout="true"  
      compression="on"  
      compressableMimeType="text/html,text/xml,text/css,text/javascript,application/x-javascript,application/javascript,image/svg+xml,text/json"/>

* Tomcat:

  in ${Tomcat_Home}/conf/server.xml, add following:

      <Connector port="8080" maxhttpheadersize="8192" maxthreads="150" minsparethreads="25" maxsparethreads="75" enablelookups="false" redirectport="8443" acceptcount="100" connectiontimeout="20000" disableuploadtimeout="true" compression="on" compressionminsize="2048" nocompressionuseragents="gozilla, traviata" compressablemimetype="text/html,text/xml">
      </Connector>


* Apache Httpd:  

  first, enable mod_deflate.so

  then add the this line in your conf:

      AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript text/json application/javascript application/xml application/json
