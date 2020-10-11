+++
title = "Static API html document"
description = "How to get static html file out of the swagger box"
date = 2020-10-11T08:13:50Z
author = "Dariusz Mozgowoj"
+++

## Introduction

*"Could you provide some api docs?"* I heard recenlty from my colegues which are working on mobile part of the application.
You can ask now... well where the problem is? We have already swagger and other tools. Unfortunately our team does not want
to expose the api docs as the web resource, so the aim was to create some static output of api docs... and this was not 
as trivial task as I assumed at the beggining.

## How to

Just for clarification the project was setup with maven and below examples provide information related to maven only.
Moreover tests run in solution was written in **Spock**, however it can be simply adapt to junit if required.

### Required repositories and dependencies
After some investigation I found that to achieve the goal requires some additional repositories and repository plugins to 
get the solution works.

```  
    <repositories>
        <repository>
            <id>spring-libs-milestone</id>
            <url>https://repo.spring.io/libs-milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <pluginRepositories>
        <pluginRepository>
            <id>spring-plugins-release</id>
            <url>https://repo.spring.io/plugins-release</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>

```
Standard swagger and swagger2markup dependencies are of course also required:

```
 <dependencies>
...
        <dependency>
            <groupId>io.swagger.codegen.v3</groupId>
            <artifactId>swagger-codegen</artifactId>
            <version>3.0.21</version>
        </dependency>
        <dependency>
            <groupId>io.github.swagger2markup</groupId>
            <artifactId>swagger2markup</artifactId>
            <version>1.3.3</version>
        </dependency>
        <dependency>
...
</dependencies>
``` 

### First step - create swagger.json
I used swagger to create json version of api documentation. To get this done a test (as presented below) was written.
Test triggered with other tests through CICD pippline each time a new portion of code is merge to master branch.

The test itself:
```
@SpringBootTest(classes = CoreApplication.class)
class GenerateSwaggerSpec extends Specification {

    @Autowired
    WebApplicationContext context

    def 'generate swagger.json file'() {
        expect:
            MockMvc mockMvc = MockMvcBuilders.webAppContextSetup(context).build()
            File swaggerFile = new File("target/generated-sources/swagger.json")
            mockMvc.perform(MockMvcRequestBuilders.get("/v2/api-docs").accept(MediaType.APPLICATION_JSON))
                    .andDo({ result -> Files.write(result.getResponse().getContentAsString().getBytes(), swaggerFile) })
    }
}
```
As presented the swagger generated data are get and written to the file predefined in the **target** directory.

### Second step - process swagger file trough swagger2markup 

Swagger.json file needs to then be processed with **swagger2markup** where markup language was in this cas ***ASCIDOC***.
Whole operation needs to be defined **build** block of pom file.
First plugin get the file generated in first step and use specified markup language (ASCIDOC here) to generate output 
in this language at location defined in plugin *../generated-sources/swagger/*
```
<build>
    <plugins>
...
            <plugin>
                <groupId>io.github.swagger2markup</groupId>
                <artifactId>swagger2markup-maven-plugin</artifactId>
                <version>1.3.3</version>
                <configuration>
                    <swaggerInput>${project.build.directory}/generated-sources/swagger.json</swaggerInput>
                    <outputDir>${project.build.directory}/generated-sources/swagger/</outputDir>
                    <config>
                        <swagger2markup.markupLanguage>ASCIIDOC</swagger2markup.markupLanguage>
                    </config>
                </configuration>
                <executions>
                    <execution>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>convertSwagger2markup</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
          
        </plugins>
    </build>
```

To get this work plugin which build ASCIDOC needs to be used too.

```
<build>
    <plugins>
...
            <plugin>
                <groupId>org.asciidoctor</groupId>
                <artifactId>asciidoctor-maven-plugin</artifactId>
                <version>1.5.3</version>
                <dependencies>
                    <dependency>
                        <groupId>org.jruby</groupId>
                        <artifactId>jruby-complete</artifactId>
                        <version>1.7.21</version>
                    </dependency>
                </dependencies>
                <configuration>
                    <sourceDirectory>${project.basedir}/src/main/resources/asciidoc/</sourceDirectory>
                    <sourceDocumentName>index.adoc</sourceDocumentName>
                    <backend>html5</backend>
                    <outputDirectory>${project.build.directory}/generated-sources/swagger-html/</outputDirectory>
                    <attributes>
                        <toc>left</toc>
                        <toclevels>3</toclevels>
                        <generated>${project.build.directory}/generated-sources/swagger/</generated>
                    </attributes>
                </configuration>
                <executions>
                    <execution>
                        <id>output-html</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>process-asciidoc</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
...
        </plugins>
    </build>
```
In presented snippet few things are going on. To get better understanding we can read snipped from buttom to top.
1. Plugin read output from swagger2markup -> ASCIDOC processing located in *.../generated-sources/swagger/*.
2. Then the file from location *.../src/main/resources/asciidoc/* called **index.adoc* is used 
3. Plugin process data and output html static file at location of *.../generated-sources/swagger-html/*

I believe that point 2 needs some more explanation 
Mentioned **index.adoc** must be specified upfront in the presented location. The structure of this file is already defined
and should look exactly as in snippet below:

```
include::{generated}/overview.adoc[]
include::{generated}/paths.adoc[]
include::{generated}/security.adoc[]
include::{generated}/definitions.adoc[]
```

### Some more details about the process 
Above structures points to files generated with swagger2markup process and they will be automatically inserted in the order 
as described in **index.adoc**.

### Hint about the CICD
In the above case **swagger.json** was generated at *test* stage of CICD pipeline. 
The static **html** file was constructed at mid-step between *test* and *deploy* stages.
The html-apidocs are available to team members at storage of the cloud solution provider.****