# Functions Framework for Go

[![GoDoc](https://godoc.org/github.com/GoogleCloudPlatform/functions-framework-go?status.svg)](http://godoc.org/github.com/GoogleCloudPlatform/functions-framework-go) [![Go version](https://img.shields.io/badge/go-v1.18+-blue)](https://golang.org/dl/#stable)

[![Go unit CI][ff_go_unit_img]][ff_go_unit_link] [![Go lint CI][ff_go_lint_img]][ff_go_lint_link] [![Go conformace CI][ff_go_conformance_img]][ff_go_conformance_link]  ![Security Scorecard](https://api.securityscorecards.dev/projects/github.com/GoogleCloudPlatform/functions-framework-go/badge)

An open source FaaS (Function as a Service) framework for writing portable
Go functions.

The Functions Framework lets you write lightweight functions that run in many
different environments, including:

*   [Google Cloud Run functions](https://cloud.google.com/functions/)
*   Your local development machine
*   [Knative](https://github.com/knative/)-based environments
*   [Google App Engine](https://cloud.google.com/appengine/docs/go/)
*   [Google Cloud Run](https://cloud.google.com/run/docs/quickstarts/build-and-deploy)

The framework allows you to go from:

```golang
func HelloWorld(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, World!")
}
```

To:

```sh
curl http://my-url
# Output: Hello, World!
```

All without needing to worry about writing an HTTP server or request
handling logic.

## Features

*   Build your Function in the same container environment used by Cloud Run functions using [buildpacks](https://github.com/GoogleCloudPlatform/buildpacks).
*   Invoke a function in response to a request
*   Automatically unmarshal events conforming to the
    [CloudEvents](https://cloudevents.io/) spec
*   Portable between serverless platforms

## Quickstart: Hello, World on your local machine

1. Install Go 1.18+.

1. Create a Go module:
	```sh
	go mod init example.com/hello
	```

	> Note: You can use a different module name rather than `example.com/hello`.

1. Create a `function.go` file with the following contents:
	```golang
	package function

	import (
		"fmt"
		"net/http"

		"github.com/GoogleCloudPlatform/functions-framework-go/functions"
	)

	func init() {
		functions.HTTP("HelloWorld", helloWorld)
	}

	// helloWorld writes "Hello, World!" to the HTTP response.
	func helloWorld(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello, World!")
	}
	```

	> Note that you can use any file name or package name (convention is to make
	package name same as directory name).

1. To run locally, you'll need to create a main package to start your server
   (see instructions below for container builds to skip this step and match your
   local development environment to production):
   ```sh
   mkdir cmd
   ```

1. Create a `cmd/main.go` file with the following contents:
	```golang
	package main

	import (
		"log"
		"os"

		// Blank-import the function package so the init() runs
		_ "example.com/hello"
		"github.com/GoogleCloudPlatform/functions-framework-go/funcframework"
	)

	func main() {
		// The local development server is normally started from the command line, but
		// certain IDE's support directly running go applications and the function won't
		// be recognized. 
		//
		// To resolve, set the "FUNCTION_TARGET" env variable directly to ensure the 
		// function is registered for the local server.
		//
		// err := os.Setenv("FUNCTION_TARGET", "HelloWorld")
		// if err != nil {
		//     log.Fatalf("failed to set FUNCTION_TARGET: %v\n for local dev server", err)
		// }
 
		// Use PORT environment variable, or default to 8080.
		port := "8080"
		if envPort := os.Getenv("PORT"); envPort != "" {
			port = envPort
		}
		
		// By default, listen on all interfaces. If testing locally, run with 
		// LOCAL_ONLY=true to avoid triggering firewall warnings and 
		// exposing the server outside of your own machine.
		hostname := ""
		if localOnly := os.Getenv("LOCAL_ONLY"); localOnly == "true" {
			hostname = "127.0.0.1"
		} 
		if err := funcframework.StartHostPort(hostname, port); err != nil {
			log.Fatalf("funcframework.StartHostPort: %v\n", err)
		}
	}
	```

1. Run `go mod tidy` to update dependency requirements.

1. Start the local development server:
	```sh
	FUNCTION_TARGET=HelloWorld LOCAL_ONLY=true go run cmd/main.go
	# Output: Serving function: HelloWorld
	```

	Upon starting, the framework will listen to HTTP requests at `/` and invoke your registered function
	specified by the `FUNCTION_TARGET` environment variable (i.e. `FUNCTION_TARGET=HelloWorld`).

1. Send requests to this function using `curl` from another terminal window:
	```sh
	curl localhost:8080
	# Output: Hello, World!
	```
## Quickstart: Enable Exeuction Id Logging

[Cloud Run Functions(1st gen)](https://cloud.google.com/functions/1stgendocs/deploy) provides an execution id in the logs at `labels.execution_id`, which customers can use to filter their logs for each execution. [Cloud Run Functions](https://cloud.google.com/functions/docs/deploy) doesn't have the same feature embedded. 

To have exeuction id logged for `Cloud Run Functions` executions, users can either:

* Provide a custom execution Id in the Http Header `Function-Execution-Id`.

	```sh
		curl -H "Function-Execution-Id: 123456" localhost:8080
		# Output: Hello, World!
	```

	Example Log:
	```
	{"message":"Try logging with executionID!","logging.googleapis.com/labels":{"execution_id":"123456"}}
	```


OR 
* Leverage `LogWriter` provided in function-framework-go(v1.9.0 or higher) library to generate logs. If `Function-Exeuction-Id` is empty, a pseduorandom execution id will be auto-generated if `LogWriter` is used.  

	```golang
	package function

	import (
		"fmt"
		"net/http"
		"log"
		"github.com/GoogleCloudPlatform/functions-framework-go/functions"
		"github.com/GoogleCloudPlatform/functions-framework-go/funcframework"
	)

	func init() {
		functions.HTTP("HelloWorld", helloWorld)
	}

	// helloWorld writes "Hello, World!" to the HTTP response.
	func helloWorld(w http.ResponseWriter, r *http.Request) {
		l := log.New(funcframework.LogWriter(r.Context()), "", 0)

		l.Println("Try logging with executionID!")
		fmt.Fprintln(w, "Hello, World!")
	}
	```

	Example Log:
	```
	{"message":"Try logging with executionID!","logging.googleapis.com/labels":{"execution_id":"181dbb5b096549313d470dd68fa64d96"}}
	```


## Go further: build a deployable container

1. Install [Docker](https://store.docker.com/search?type=edition&offering=community) and the [`pack` tool](https://buildpacks.io/docs/install-pack/).

1. Build a container from your function using the Functions [buildpacks](https://github.com/GoogleCloudPlatform/buildpacks):
	```sh
	pack build \
		--builder gcr.io/buildpacks/builder:v1 \
		--env GOOGLE_FUNCTION_SIGNATURE_TYPE=http \
		--env GOOGLE_FUNCTION_TARGET=HelloWorld \
		my-first-function
	```

1. Start the built container:
	```sh
	docker run --rm -p 8080:8080 my-first-function
	# Output: Serving function...
	```

1. Send requests to this function using `curl` from another terminal window:
	```sh
	curl localhost:8080
	# Output: Hello, World!
	```

## Run your function on serverless platforms

### Google Cloud Run functions

Deploy from your local machine using the `gcloud` command-line tool.
[Check out the Cloud Functions quickstart](https://cloud.google.com/functions/docs/quickstart).

### Container environments based on Knative

The Functions Framework is designed to be compatible with Knative environments.
Just build and deploy your container to a Knative environment. Note that your app needs to listen
`PORT` environment variable per [Knative runtime contract](https://github.com/knative/serving/blob/main/docs/runtime-contract.md#inbound-network-connectivity).

## Functions Framework Features

The Go Functions Framework conforms to the [Functions Framework Contract](https://github.com/GoogleCloudPlatform/functions-framework), As such, it
supports HTTP functions, background event functions, and CloudEvent functions
(as of v1.1.0). The primary build mechanism is the [GCP buildpacks stack](https://github.com/GoogleCloudPlatform/buildpacks), which takes a function of
one of the accepted types, converts it to a full HTTP serving app, and creates a
launchable container to run the server.

### HTTP Functions

The Framework provides support for handling native Go HTTP-style functions:

```golang
package function

import (
	"net/http"

	"github.com/GoogleCloudPlatform/functions-framework-go/functions"
)

func init() {
	functions.HTTP("HelloWorld", helloWorld)
}

func helloWorld(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, World!"))
}
```

### CloudEvent Functions

The Functions Framework provides support for unmarshalling an incoming
[CloudEvent](https://cloudevents.io/) payload into a `cloudevents.Event` object.
These will be passed as arguments to your function when it receives a request.

```golang
package function

import (
	cloudevents "github.com/cloudevents/sdk-go/v2"
	"github.com/GoogleCloudPlatform/functions-framework-go/functions"
)

func init() {
	functions.CloudEvent("CloudEventFunc", cloudEventFunc)
}

func cloudEventFunc(ctx context.Context, e cloudevents.Event) error {
	// Do something with event.Context and event.Data (via event.DataAs(foo)).
	return nil
}
```

These functions are registered with the handler via `funcframework.RegisterCloudEventFunctionContext`.

To learn more about CloudEvents, see the [Go SDK for CloudEvents](https://github.com/cloudevents/sdk-go).

### Background Event Functions

[Background events](https://cloud.google.com/functions/docs/writing/background)
are also supported. This type of function takes two parameters: a Go context and
a user-defined data struct.

```golang
func BackgroundEventFunction(ctx context.Context, data userDefinedEventStruct) error {
	// Do something with ctx and data.
}
```

This type of event requires you to define a struct with the
appropriate data fields (e.g. those for a PubSub message or GCS event) and pass
that struct as the data parameter. See the [samples](https://cloud.google.com/functions/docs/writing/background) for details.

The context parameter is a Go `context.Context`, and contains additional event
metadata under a functions-specific key. This data is accesible via the `cloud.google.com/go/functions/metadata` package:

```golang
m := metadata.FromContext(ctx)
```

These functions can be registered in `main.go` for local testing with the handler via `funcframework.RegisterEventFunctionContext`.

[ff_go_unit_img]: https://github.com/GoogleCloudPlatform/functions-framework-go/workflows/Go%20Unit%20CI/badge.svg
[ff_go_unit_link]: https://github.com/GoogleCloudPlatform/functions-framework-go/actions?query=workflow%3A"Go+Unit+CI"
[ff_go_lint_img]: https://github.com/GoogleCloudPlatform/functions-framework-go/workflows/Go%20Lint%20CI/badge.svg
[ff_go_lint_link]: https://github.com/GoogleCloudPlatform/functions-framework-go/actions?query=workflow%3A"Go+Lint+CI"
[ff_go_conformance_img]: https://github.com/GoogleCloudPlatform/functions-framework-go/workflows/Go%20Conformance%20CI/badge.svg
[ff_go_conformance_link]: https://github.com/GoogleCloudPlatform/functions-framework-go/actions?query=workflow%3A"Go+Conformance+CI"
