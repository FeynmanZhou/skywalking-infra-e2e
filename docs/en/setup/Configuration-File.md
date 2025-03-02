# SkyWalking Infra E2E Configuration Guide

The configuration file is used to integrate all the step configuration content.
You can see the sample configuration files for different environments in the [examples directory](../../../examples).

There is a quick view about the configuration file, and using the `yaml` format.
```yaml
setup:
  # set up the environment
cleanup:
  # clean up the environment
trigger:
  # generate traffic
verify:
  # test cases
```

## Setup

Support two kinds of the environment to set up the system.

### KinD

```yaml
setup:
  env: kind
  file: path/to/kind.yaml  # Specified kinD manifest file path
  timeout: 1200            # timeout second
  steps:                   # customize steps for prepare the environment
    - name: customize setups        # step name
      # one of command line or kinD manifest file
      command: command lines        # use command line to setup 
      path: /path/to/manifest.yaml  # the manifest file path
      wait:                         # how to verify the manifest is set up finish
        - namespace:                # The pod namespace
          resource:                 # The pod resource name
          label-selector:           # The resource label selector
          for:                      # The wait condition 
```

The `KinD` environment follow these steps:
1. Start the `KinD` cluster according to the config file.
1. Apply the resources files (`--manifests`) or/and run the custom init command (`--commands`) by steps.
1. Wait until all steps are finished and all services are ready with the timeout(second).

### Compose

```yaml
setup:
  env: compose
  file: path/to/compose.yaml  # Specified docker-compose file path
  timeout: 1200               # Timeout second
  steps:                      # Customize steps for prepare the environment
    - name: customize setups        # Step name
      command: command lines        # Use command line to setup 
```

The `docker-compose` environment follow these steps:
1. Start the `docker-compose` services.
1. Check the services' healthiness.
1. Wait until all services are ready according to the interval, etc.
1. Execute command to set up the testing environment or help verify.

#### Service Export
If you want to get the service host and port mapping, should follow these steps:
1. declare the port in the `docker-compose` service `ports` config.
   ```yaml
   oap:
    image: xx.xx:1.0.0
    ports:
        # define the port
        - 8080
   ```
1. Follow this format to get the host and port mapping by the environment, and it's available in steps(trigger, verify).
   ```yaml
   trigger:
      # trigger with specified mappinged port
      url: http://${oap.host}:${oap_8080}/
   ```

## Trigger

After the `Setup` step is finished, use the `Trigger` step to generate traffic.

```yaml
trigger:
  action: http      # The action of the trigger. support HTTP invoke.
  interval: 3s      # Trigger the action every 3 seconds.
  times: 5          # How many times to trigger the action, 0=infinite.
  url: http://apache.skywalking.com/ # Http trigger url link.
  method: GET       # Http trigger method.
```

The Trigger executed successfully at least once, after success, the next stage could be continued. Otherwise, there is an error and exit.

## Verify

After the `Trigger` step is finished, running test cases.

```yaml
verify:
  retry:            # verify with retry strategy
    count: 10       # max retry count
    interval: 10000 # the interval between two retries, in millisecond.
  cases:            # verify test cases
    - actual: path/to/actual.yaml       # verify by actual file path
      expected: path/to/expected.yaml   # excepted content file path
    - query: echo 'foo'                 # verify by command execute output
      expected: path/to/expected.yaml   # excepted content file path
```

The test cases are executed in the order of declaration from top to bottom, If the execution fails, and the retry strategy is exceeded, the process is deemed to have failed.

### Retry strategy

The retry strategy could retry automatically on the test case failure, and restart by the failed test case.

### Case source

Support two kind source to verify, one case only supports one kind source type:

1. source file: verify by generated `yaml` format file.
2. command: use command line output as they need to verify content, also only support `yaml` format.

### Excepted verify template

After clarifying the content that needs to be verified, you need to write content to verify the real content and ensure that the data is correct.

You need to use the form of [Go Template](https://pkg.go.dev/text/template#pkg-overview) to write the verification file, and the data content to be rendered comes from the real data. By verifying whether the rendered data is consistent with the real data, it is verified whether the content is consistent.
You could see [many test cases in this directory](../../../test/verify).

We have done a lot of extension functions for verification functions on the original Go Template.

#### Extension functions

Extension functions are used to help users quickly locate the problem content and write test cases that are easier to use.

##### Basic Matches

Verify that the number fits the range.

|Function|Description|Grammar|Verify success|Verify failure|
|-------|------------|-------|-------------|-------------|
|gt|Verify the first param is greater than second param |{{gt param1 param2}}|param1|<wanted gt $param2, but was $param1>|
|ge|Verify the first param is greater than or equals second param |{{ge param1 param2}}|param1|<wanted gt $param2, but was $param1>|
|lt|Verify the first param is less than second param |{{lt param1 param2}}|param1|<wanted gt $param2, but was $param1>|
|le|Verify the first param is less than or equals second param |{{le param1 param2}}|param1|<wanted gt $param2, but was $param1>|
|regexp|Verify the first param matches the second regular expression|{{regexp param1 param2}}|param1|<"$param1" does not match the pattern $param2">|
|notEmpty|Verify The param is not empty|{{notEmpty param}}|param|<"" is empty, wanted is not empty>|

##### List Matches

Verify the data in the condition list, Currently, it is only supported when all the conditions in the list are executed, it is considered as successful.

Here is an example, It's means the list values must have value is greater than 0, also have value greater than 1, Otherwise verify is failure.
```yaml
{{- contains .list }}
- key: {{ gt .value 0 }}
- key: {{ gt .value 1 }}
        {{- end }}
```

##### Encoding

In order to make the program easier for users to read and use, some code conversions are provided.

|Function|Description|Grammar|Result|
|-------|------------|-------|------|
|b64enc|Base64 encode|{{ b64enc "Foo" }}|Zm9v|

## Cleanup

After the E2E finished, how to clean up the environment.

```yaml
cleanup:
   on: always     # Clean up strategy
```

Supports the following strategies:
1. `always`: No matter the execution result is success or failure, cleanup will be performed.
1. `success`: Only when the execution succeeds.
1. `failure`: Only when the execution failed.
1. `never`: Never clean up the environment.

