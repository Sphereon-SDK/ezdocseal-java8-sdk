# Using the EzDocSeal Java SDK
Before proceeding, install a [JDK](https://docs.oracle.com/javase/8/docs/technotes/guides/install/install_overview.html) (must be Java 8 or later) and [Apache Maven](https://maven.apache.org/install.html).

Ensure `JAVA_HOME` is set correctly and the `mvn` executable is available on your PATH.

There are two types of SDKs available.
1. Only the client library of the SDK, which requires additional libraries as dependency
2. A "fat jar" that contains all required dependencies; No additional dependencies are needed.


### Maven resolution

To depend on this project in Apache Maven, add the following to your pom.xml file (the dots are just to highlight that there typically is more text at these locations).
```xml
<!-- Existing pom.xml config -->

<repositories>

    <!-- If you want to use released versions of the SDK -->
    <repository>
        <id>sphereon-sdk-releases</id>
        <url>https://nexus.qa.sphereon.com/repository/sphereon-sdk-releases/</url>
    </repository>

    <!-- If you want to use -SNAPSHOT (development) versions of the SDK -->
    <repository>
        <id>sphereon-sdk-snapshots</id>
        <url>https://nexus.qa.sphereon.com/repository/sphereon-sdk-snapshots/</url>
    </repository>

    <!-- Existing repositories -->
</repositories>

<!-- Existing pom.xml config -->

<dependencies>
    <dependency>
        <groupId>com.sphereon.sdk</groupId>
        <artifactId>ezdocseal-java8-sdk</artifactId>
        <!-- Snapshot/development version, remove -SNAPSHOT for releases -->
        <version>1.0.0-SNAPSHOT</version>
    </dependency>
    
    <!-- Existing dependencies -->
</dependencies>
  
<!-- Existing pom.xml config -->
```

### Gradle resolution
To depend on this project in Gradle instead of maven, add the following to your build.gradle file.
```perl
dependencies {
    compile 'com.sphereon.sdk:ezdocseal-java8-sdk:1.0.0-SNAPSHOT'
}
```



# Developer Guide

## Creating a client
  To create a client, use the client builder. You can obtain an instance of the builder via a static factory method located on the client interface.

```java
  EzDocSealClientBuilder builder = EzDocSeal.builder();
```

  The builder exposes many fluent configuration methods that can be chained to configure a service client. Here's a simple example that sets a few optional configuration options and then builds the service client.
```java
EzDocSeal client = EzDocSeal.builder()
  .connectionConfiguration(new ConnectionConfiguration()
    .maxConnections(100)
    .connectionMaxIdleMillis(1000))
  .timeoutConfiguration(new TimeoutConfiguration()
    .httpRequestTimeout(3000)
    .totalExecutionTimeout(10000)
    .socketTimeout(2000))
  .build();
```

### Client Lifecycle
Clients clean up their resources when garbage collected but if you want to explicitly shut down the client you can do the following:
```java
EzDocSeal client = EzDocSeal.builder().build();
  client.shutdown();
  // Client is now unusable
```
## API keys
  An API key must be provided by the service owner. After they key is available, it can be set via the client builder. It is recommended to treat the API key as sensitive and not hard-code it in your source code.

```java
EzDocSeal client = EzDocSeal.builder()
  .apiKey("customers-api-key")
  .build();
```

  After it is configured, the API key is sent with every request made to the service via the `x-api-key` header.

  See [http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-api-keys.html](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-api-keys.html) for more information.



## Complete example to sign a PDF

````java
 public void example(File inputFile) throws IOException, URISyntaxException {
        final byte[] inputPdfBytes = Files.readAllBytes(inputFile.toPath());
        final String pdfContent = Base64.encodeAsString(inputPdfBytes);


        // Create the client
        EzDocSeal client = EzDocSeal.builder()
          // The API Key. Replace with your API key!
          .apiKey("<api-key>")

          // Connection configuration
          .connectionConfiguration(new ConnectionConfiguration()
            .maxConnections(100)
            .connectionMaxIdleMillis(1000))

          // Timeouts
          .timeoutConfiguration(new TimeoutConfiguration()
            .httpRequestTimeout(3000)
            .totalExecutionTimeout(10000)
            .socketTimeout(2000))
          .build();


        // Create the actual sign request body
        final SignRequest signRequest = new SignRequest().jsonSignRequest(
          new JsonSignRequest().signData(
            new SignData()
              .emailAddress("test@example.com")
              .name("Bob the Builder")
              .location("New York, USA")
              .reason("Approved")
          ).content(
            pdfContent
          )
        );

        // Call the sign API
        final SignResult result = client.sign(signRequest);

        // Base64 decode the returned content that contains the signed PDF
        final byte[] signedPdfBytes = Base64.decode(result.getJsonSignResponse().getContent());

        // Write the PDF data
        Files.write(new File("signed-example-output.pdf").toPath(), signedPdfBytes);

    }
````

## Making requests
  After a client is configured and created, you can make a request to the service. A method on the client interface (`EzDocSeal`) is created for all actions (resource + method) in your API.

  For each API method, classes are generated that represent the request and response of that API. The request class has setters for any path parameters, query parameters, headers, and payload model that are defined in the API. The response class exposes getters for any modeled headers and for the modeled payload.
```java
  EzDocSeal client = EzDocSeal.builder().build();
  SignResult result = client.sign(new SignRequest());
```

### Request Configuration
  In addition to client-level configuration configured by the builder, each request class exposes configuration methods that are scoped to that request alone. Request level configuration takes precedence over client level configuration.

  The request config also allows adding headers and query parameters that aren't modeled by the API.

```java
EzDocSeal client = EzDocSeal.builder().build();
client.sign(new SignRequest().sdkRequestConfig(
  SdkRequestConfig.builder()
    .httpRequestTimeout(1500)
    .totalExecutionTimeout(5000)
    .customHeader("CustomHeaderName", "foo")
    .customQueryParam("CustomQueryParamName", "bar")
    .build()
  ));
```

### Response Metadata
  In addition to the modeled data present in result objects, the SDK exposes access to additional HTTP metadata. This metadata is useful for debugging issues or accessing unmodeled data from the HTTP response.
```java
  SignResult result = client.sign(new SignRequest());
  System.out.println(result.sdkResponseMetadata().requestId());
  System.out.println(result.sdkResponseMetadata().httpStatusCode());
  // Full access to all HTTP headers (including modeled ones)
  result.sdkResponseMetadata().header("Content-Length").ifPresent(System.out::println);
```

## Exception Handling

  Service exceptions and client exceptions can be handled separately. Any exceptions modeled in the API will be a subtype of EzDocSealException.
```java
  try {
    client.sign(...);
  } catch(BadRequestException e) {
    // This is a modeled exception defined in the API
  } catch(EzDocSealException e) {
    // All service exceptions will extend from EzDocSealException.
    // Any unknown or unmodeled service exceptions will be represented as a EzDocSealException.
  } catch(SdkClientException e) {
    // Client exceptions include timeouts, IOExceptions, or any other exceptional situation where a response
    // is not received from the service.
  }
```

  All exceptions that can be thrown by the SDK are a subtype of SdkBaseException. To handle any exception in the same way, you can directly catch this exception. This covers both client and service exceptions.
```java
  try {
    client.sign(...);
  } catch(SdkBaseException e) {
    // All exceptions thrown from the client will be a subtype of SdkBaseException.
  }
```

  All service exceptions expose metadata about the HTTP response for logging or debugging purposes.
```java
  try {
    client.sign(...);
  } catch(EzDocSealException e) {
    int statusCode = e.sdkHttpMetadata().httpStatusCode();
    String requestId = e.sdkHttpMetadata().requestId();
    Optional<String> contentLength = e.sdkHttpMetadata().header("Content-Length");
    ByteBuffer responseContent = e.sdkHttpMetadata().responseContent();
  }
```

  Some client exceptions thrown are subtypes of SdkClientException. This provides greater granularity to deal with client-side exceptions.
```java
  try {
    client.sign(...);
  } catch(ClientExecutionTimeoutException e) {
    // Specific client exception thrown when the totalExecutionTimeout is triggered.
  } catch(AbortedException e) {
    // Thrown when the client thread is interrupted while making a request.
  } catch(SdkClientException e) {
    // All other exceptions can be handled here.
  }
```

## Retries
  Out of the box, the generated client retries on throttling errors (HTTP status code 429) and connection exceptions. If a different retry policy is desired, a custom one can be set via the client builder.

  The easiest way to create a custom retry policy is to use the RetryPolicyBuilder. It provides a declarative API to specify when to retry.

```java
  /**
* The policy below will retry if the cause of the failed request matches any of the exceptions
* given OR if the HTTP response from the service has one of the provided status codes.
  */
EzDocSeal client = EzDocSeal.builder()
  .retryPolicy(RetryPolicyBuilder.standard()
    .retryOnExceptions(BadRequestException.class, SocketTimeoutException.class)
    .retryOnStatusCodes(429, 500)
    .maxNumberOfRetries(10)
    .fixedBackoff(100)
    .build())
  .build();
```

  You can also directly implement the RetryPolicy interface to define your own implementation. RetryPolicyContext contains useful metadata about the state of the failed request that can be used to drive dynamic retry decisions or compute backoff delays.

```java
  /**
* Simple implementation of {@link com.amazonaws.retry.v2.RetryPolicy}
  */
public static class CustomRetryPolicy implements RetryPolicy {

  @Override
  public long computeDelayBeforeNextRetry(RetryPolicyContext context) {
      return 100;
  }

  @Override
  public boolean shouldRetry(RetryPolicyContext context) {
    return context.retriesAttempted() < 3;
  }
}

// Using a custom retry policy via the builder
EzDocSeal client = EzDocSeal.builder()
    .retryPolicy(new CustomRetryPolicy())
    .build();
```

  You can implement a RetryCondition and BackoffStrategy separately and combine them into a single policy.

```java
/**
* Retry on 429 status codes. It's important to note that status code may be null if no response was returned from the
* service. See {@link com.amazonaws.retry.v2.RetryCondition}
*/
public static class RetryOnThrottlingCondition implements RetryCondition {

  @Override
  public boolean shouldRetry(RetryPolicyContext context) {
      return context.httpStatusCode() != null && context.httpStatusCode() == 429;
  }
}

/**
* Simple implementation of {@link BackoffStrategy} that always backs off 100 milliseconds.
*/
public static class Backoff100MillisecondsStrategy implements BackoffStrategy {
  @Override
  public long computeDelayBeforeNextRetry(RetryPolicyContext context) {
      return 100;
  }
}

/**
* Uses {@link com.amazonaws.retry.v2.SimpleRetryPolicy} to combine a separately implemented RetryCondition and BackoffStrategy.
*/
EzDocSeal client = EzDocSeal.builder()
  .retryPolicy(new SimpleRetryPolicy(new RetryOnThrottlingCondition(), new Backoff100MillisecondsStrategy()))
  .build();

/**
* The RetryCondition and BackoffStrategy interfaces are functional interfaces so lambda expressions and method
* references may also be used. This is equivalent to the above example.
*/
EzDocSeal client = EzDocSeal.builder()
  .retryPolicy(new SimpleRetryPolicy(c -> c.httpStatusCode() != null && c.httpStatusCode() == 429,
  c -> 100))
  .build();
```


## 1. Building the code yourself instead of using a released maven artifact:
```bash
      mvn package
```

This compiles the client into a jar located at `./target/ez-docseal-sdk-1.0.0-SNAPSHOT.jar`. Note that this jar does not include any dependencies.
## 2. Building a Standalone Jar

If your customers aren't using Maven or Gradle and you would prefer to distribute a single JAR with all dependencies, you can use the following command to build a fat jar.

```bash
mvn clean package -Pstandalone-jar
```

This will compile the client and package all dependencies into a jar located at `./target/ez-docseal-sdk-1.0.0-SNAPSHOT.jar`.
For more information on managing dependencies with Maven and publishing artifacts, see:
* [https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
* [http://central.sonatype.org/pages/ossrh-guide.html](http://central.sonatype.org/pages/ossrh-guide.html)
  Run the following command in a terminal/console.
  Note that each client created has its own connection pool. It's recommended to treat the clients as long-lived objects. Clients are immutable and thread safe.
