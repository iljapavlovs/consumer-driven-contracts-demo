
### On the Producer Side
1. add files with REST or messaging contracts expressed in either Groovy DSL or YAML to the contracts directory, which is set by the `contractsDslDir` property. By default, it is 
```
$rootDir/src/test/resources/contracts
```
2. Then you can add the Spring Cloud Contract **Verifier dependency and plugin** to your build file, as the following example shows:
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>
```
The following listing shows how to add the plugin, which should go in the `build/plugins` portion of the file:
```
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>${spring-cloud-contract.version}</version>
    <extensions>true</extensions>
</plugin>
```
3. Running `./mvnw clean install` automatically generates tests that verify the application compliance with the added contracts. By default, the tests get generated under `org.springframework.cloud.contract.verifier.tests.`

> (When we run the build, the plugin automatically generates a test class named ContractVerifierTest that extends our BaseTestClass and puts it in `/target/generated-test-sources/contracts/`)

As the implementation of the functionalities described by the contracts is not yet present, the tests fail.

4. add the correct implementation of either handling HTTP requests or messages
5. Also, you must add a base test class for auto-generated tests to the project. This class is extended by all the auto-generated tests, and it should contain all the setup information necessary to run them (for example RestAssuredMockMvc controller setup or messaging test setup).
   
   The following example, from pom.xml, shows how to specify the base test class:
```   
   <build>
       <plugins>
           <plugin>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-contract-maven-plugin</artifactId>
               <version>${spring-cloud-contract.version}</version>
               <extensions>true</extensions>
               <configuration>
                   <baseClassForTests>com.example.contractTest.BaseTestClass</baseClassForTests>
               </configuration>
           </plugin>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
           </plugin>
       </plugins>
   </build>
```

6. Once the 
* implementation and the test base class are in place, 
* the tests pass, and 
* both the application and the stub artifacts are built and installed in the local Maven repository.

You can now merge the changes, and you can **publish both the application and the stub artifacts in an online repository.** 


### On the Consumer Side

You can use Spring Cloud Contract **Stub Runner** in the integration tests to get a running WireMock instance or messaging route that simulates the actual service.

To do so, add the dependency to Spring Cloud Contract Stub Runner, as the following example shows:

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
You can get the Producer-side stubs installed in your Maven repository in either of two ways:

* By checking out the Producer side repository and adding contracts and generating the stubs by running the following commands:
```
$ cd local-http-server-repo
$ ./mvnw clean install -DskipTests
```
The tests are being skipped because the producer-side contract implementation is not in place yet, so the automatically-generated contract tests fail.

* By getting already-existing producer service stubs from a remote repository. To do so, pass the stub artifact IDs and artifact repository URL as Spring Cloud Contract Stub Runner properties, as the following example shows:
```
    stubrunner:
      ids: 'com.example:http-server-dsl:+:stubs:8080'
      repositoryRoot: https://repo.spring.io/libs-snapshot
```
Now you can annotate your test class with @AutoConfigureStubRunner. In the annotation, provide the group-id and artifact-id values for Spring Cloud Contract Stub Runner to run the collaborators' stubs for you, as the following example shows:
```
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment=WebEnvironment.NONE)
@AutoConfigureStubRunner(ids = {"com.example:http-server-dsl:+:stubs:6565"},
        stubsMode = StubRunnerProperties.StubsMode.LOCAL)
public class LoanApplicationServiceTests {
```
Use the REMOTE stubsMode when downloading stubs from an online repository and LOCAL for offline work.

Now, in your integration test, you can receive stubbed versions of HTTP responses or messages that are expected to be emitted by the collaborator service.





* Producer side needs _**spring-cloud-starter-contract-verifier**_ dependency
* 
