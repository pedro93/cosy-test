# cosy-test
[![Build Status](https://travis-ci.org/feedzai/cosy-test.svg?branch=master)](https://travis-ci.org/feedzai/cosy-test)
[![codecov](https://codecov.io/gh/feedzai/cosy-test/branch/master/graph/badge.svg)](https://codecov.io/gh/feedzai/cosy-test)

Bringing up Docker Compose environments for system, integration and performance testing, with support for [ScalaTest](http://www.scalatest.org/),
[JUnit](https://junit.org/junit4/) and [Gatling](https://gatling.io/).

## Why do I need this?

Imagine you need to test a complex system that requires having a lot of components working and simulating a realistic scenario.
Probably, you would just use a docker-compose file to start your environment and then do your tests.
That seems easy... But:

* How would you know that the environment is already up and running for testing?
* What if you need to test a lot of those environments concurrently?
* Would you be capable of managing all the container mapped ports without conflicts?

That is where `cosy-test` can make your life easier.

## What is cosy-test and how it helps you?

It is a simple framework that allows integration with `ScalaTest`, `JUnit` and `Gatling` testing frameworks.
With `cosy-test` it is possible to simply use docker compose files and define environment variables in order to
start docker environments, run tests and bring environments down without pains and restrictions.

It helps you by:

* Bringing up before tests, the docker environments defined in docker compose files and bringing it down afterward.
* Start tests just after all container health checks are valid.
* Get container logs for debugging purposes.
* Giving possibility of keeping containers on test failure or success.
* Control your system through environment variables.
* Providing container ids and port mappings.
* Having no [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) version dependencies.
* Not adding restrictions for testing parallelism implementation.

## Requirements

In order to `cosy-test` work, it is necessary to have installed Docker and Docker Compose. There are no version restrictions,
however we recommend using:

- Docker >= 18.02.0 (_older versions have problems during containers start up and/or tear down_)
- Docker Compose >= 1.17.1

## Usage

SBT

    libraryDependencies += "com.feedzai" %% "cosy-test" % "0.0.3"

Maven

    <dependency>
        <groupId>com.feedzai</groupId>
        <artifactId>cosy-test_2.12</artifactId>
        <version>0.0.3</version>
    </dependency>

## Example - ScalaTest

``` scala
class IntegrationSpec extends FlatSpec with DockerComposeTestSuite with MustMatchers {

  override def dockerSetup = Some(
    DockerComposeSetup(
      "scalatest",
      Seq(Paths.get("src", "test", "resources", "docker-compose-scalatest.yml")),
      Paths.get("").toAbsolutePath,
      Map.empty
    )
  )

  behavior of "Scala Test"

  it must "Retrieve all services" in {
    val expectedServices = Set("container1", "container2", "container3")
    dockerSetup.foreach { setup =>
      setup.getServices().toSet mustEqual expectedServices
    }
  }
}
```


## Example - JUnit

``` java
public class IntegrationSpec {

    private static final DockerComposeJavaSetup dockerSetup;

    static {
        dockerSetup = new DockerComposeJavaSetup(
            "junittest",
            Collections.singletonList(Paths.get("src", "test", "resources", "docker-compose-junit.yml")),
            Paths.get("").toAbsolutePath(),
            new HashMap<>()
        );
    }

    @ClassRule
    public static DockerComposeRule dockerComposeRule = new DockerComposeRule(dockerSetup);

    @Rule
    public TestWatcher testWatcher = new TestWatcher() {
        @Override
        protected void failed(Throwable e, Description description) {
            dockerComposeRule.setTestFailed(true);
        }
    };

    @Test
    public void fetchServices() {
       Assert.assertThat(
           dockerSetup.getServices(),
           containsInAnyOrder("container1", "container2", "container3")
       );
    }
}
```


## Example - Gatling

``` scala
class IntegrationSpec extends DockerComposeSimulation {

  override def dockerSetup = Some(
    DockerComposeSetup(
      "gatling",
      Seq(Paths.get("src", "test", "resources", "docker-compose-gatling.yml")),
      Paths.get("").toAbsolutePath,
      Map.empty
    )
  )

  beforeSimulation()

  private val HttpProtocol: HttpProtocolBuilder = http
    .baseURL("http://localhost:8086")

  private val populationBuilder = {
    val getAction = http("Gatling simulation").get("")
    val s = scenario("Gatling simulation")
    s.during(10.seconds)(feed(Iterator.empty).exec(getAction))
      .inject(rampUsers(10).over(1.second))
      .protocols(HttpProtocol)
  }

  setUp(populationBuilder)
}
```

## Build, Test and Package

After cloning the repository you can simply build and run the tests, by executing the command:

        sbt test

And to generate binaries:

        sbt package
