# Sandbox: Example Application

This repository contains an example backend app (oracle, API, and chaincode).

```
                     FE Portal
                        +
                        |
         +--------------v---------------+
         |                              +<----+ Swagger Specification:
         |        Middleware API        |       api/swagger/oracle.swagger.json
         +--------------+---------------+
         |  Middleware Oracle Service   |
         |  oracleserv/sandbox-oracle/  |
         +------------------+-----------+
                            |
                   JSON-RPC |
               +------------v-----------+
               |  shiroclient gateway   |
               |  substrate/shiroclient |
               +-------------+----------+
                             |
                             | JSON-RPC
 +---------------------------v--------------------------+
 |                   Phylum Business Logic              |
 |                    phylum/                           |
 +------------------------------------------------------+
 |       Substrate Chaincode (Smart Contract Runtime)   |
 +------------------------------------------------------+
 |            Hyperledger Fabric Services               |
 +------------------------------------------------------+
```

This repo includes an end-to-end "hello world" application described below.

## Luther Documentation
Check out the [docs](https://docs.luthersystems.com).

## Getting Started

*IMPORTANT:* Place your license key in `~/.luther-license.yaml`.

Ensure you have the build dependencies. On MacOS you can use the commands, using [homebrew](https://brew.sh/):

```
brew install make git go wget jq
brew install --cask docker
```

*IMPORTANT:* Make sure your `docker --version` is  >= 20.10.6.

If you are not using `brew`, make sure xcode tools are installed:

```
xcode-select --install
```

Clone this repo:

```
git clone https://github.com/luthersystems/sandbox.git
```

Run `make` to build all the services:

```
make
```

### Running the Application

First we'll run the sample application with a local instance of the LEIA
platform (gateway, chaincode, and a fabric network).  Run `make up` to bring up
a local docker network running the application and platform containers.

```
make up
```

After this completes successfully run `docker ps` which lists the running
containers.  The REST/JSON API is accessible from your localhost on port 8080
which can be spot-tested using cURL and jq:

```
curl -v http://localhost:8080/v1/health_check | jq .
```

With the containers running we can also run the end-to-end integration tests.
Once the tests complete `make down` will cleanup all the containers.

```
make integration
make down
```

Running `docker ps` again will show all the containers have been removed.

## "Hello World" Application

This repo includes a small application for managing account balances. It serves
a JSON API that provides endpoints to:

1) create an account with a balance
2) look up the balance for an account
3) transfer between two accounts

> To simplify the sandbox, we have omitted authentication which we handle
> using [lutherauth](https://docs.luthersystems.com/luther/application/modules/lutherauth).
> Authorization is implemented at the application layer over tokens issued by
> lutherauth.

### Directory Structure

Overview of the directory structure

```	
build/:
	Temporary build artifacts (do not check into git).
common.config.mk:
	User-defined settings & overrides across the project.
api/:
	API specification and artifacts. See README.
compose/:
	Configuration for docker compose networks that are brought up during
	testing. These configurations are used by the existing Make targets
	and `sandbox_compose.py`.
fabric/:
	Configuration and scripts to launch a fabric network locally.
oracleserv/sandbox-oracle/:
	The oracle service responsible for serving the REST/JSON APIs and
	communicating with other microservices.
phylum/:
	Business logic that is executed "on-chain" using the platform (substrate).
scripts/:
	Helper scripts for the build process.
tests/:
	End-to-end API tests that use martin.
```

### Developing the application

The API is defined using protobuf objects and service definitions under the
`api/` directory.  Learn more about how the API is defined and the data model
definitions by reading the sandbox API's [documentation](api/).

The application API is served by the "oracle", which interfaces with the LEIA
platform.  Learn more about the design of the oracle and how to extend its
functionality by reading the sandbox oracle's
[documentation](oracleserv/sandbox-oracle/).

The oracle interacts with the core business logic that is defined by the
"phylum", [elps](https://github.com/luthersystems/elps) code that defines an
application's business rules.  Learn more about writing phyla by reading the
sandbox phylum's [documentation](phylum/).

### Testing Modifications

There are 3 main types of tests in this project:

1) Phylum _unit_ tests.  These tests excercise busines rules and logic around
   storage of smart contract data model entities.  More information about
   writing and running unit tests can be found in the phylum
   [documentation](phylum/).

2) Oracle _functional_ tests.  These tests exercise API endpoints and their
   connectivity to the phylum application layer.  More information about writing
   and running functional tests can be found in the oracle
   [documentation](oracleserv/sandbox-oracle/).

3) End-To-End _integration_ tests.  These tests use the `martin` tool.  These
   tests exercise realistic end-user functionality of the oracle REST/JSON APIs
   using [Postman](https://www.postman.com/product/api-client/) under the hood.
   More information about writing and running integration tests can be found in
   the test [documentation](tests/)

After making some changes to the phylum's business logic, the oracle middleware,
or the API it is a good idea to test those changes.  The quickest integrity
check to detect errors in the application is to run the phylum unit tests and
API functional tests from the phylum and oracle directories respectively.  This
can be done easily from the application's top level with the following command:

```
make test
```

Instead of running the above command the phylum and oracle can be tested
individually with the following commands:

```
make phylumtest
make oraclegotest
```

If these tests pass then one can move on to run the end-to-end integration tests
against a real network of docker containers.  As done in the Getting Started
section, this will require running `make up` to create a network and `make
integration` to actually run the tests.

```
make up
make integration
```

### Over-The-Air (OTA) Update

During development application, particularly if developing an application with a
UI, phylum bugs may be discovered while the application is running (i.e. `make
up`).  After fixing bugs in the local phylum code, redeploy the code onto the
running fabric network with the following shell command:

```
(cd fabric && make init)
```

This uses the OTA Update module to immediately install the new business logic on
to the fabric network.  The upgrade here is done the same way devops engineers
would perform the application upgrade when running the platform on production
infrastructure.

### Rapid Development and Live Reloading in an In-Memory Simulation

Constantly running a local instance of the LEIA platform can consume a lot of
computer resources and running `make up` and `make down` frequently is time
consuming.  Instead of running the complete platform it can be simulated
locally, in-memory.  Running an in-memory version of the platform is much faster
and less resource intensive.  In contrast to running the real platform which is
done with `make up` running the application with an in-memory platform is done
with the `make mem-up` command.

```
make mem-up
make integration
```

Running `docker ps` at this point will show that only the application
oracle/middleware is running.  Beyond starting fast and consuming fewer
resources, the in-memory platform also features *live code reloading* where any
phylum code changes will immediately be reflected in the running application.

If integration tests fail after making modifications you can diagnose them by
reading the test output and comparing that with the application logs which are
found by running the following command:

```
docker logs sandbox_oracle
```

As with running the real platform, the oracle docker container and the in-memory
platform are cleaned up by running the command:

```
make down
```
