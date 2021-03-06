# CF Acceptance Tests (CATs)

This test suite exercises a full [Cloud Foundry](https://github.com/cloudfoundry/cf-release)
deployment using the golang `cf` CLI and `curl`.
It is restricted to testing user-facing features such as a user interacting with the system via the CLI.

For example, one test pushes an app with `cf push`, hits an endpoint on the app
with `curl` that causes it to crash, and asserts that we eventually see a crash
event registered in `cf events`.

Tests that will NOT be introduced here are ones which could be tested at the component level,
such as basic CRUD of an object in the Cloud Controller. These tests belong with that component.

These tests are not intended for use against production systems,
they are intended for acceptance environments for teams developing Cloud Foundry itself.
While these tests attempt to clean up after themselves,
there is no guarantee that they will not mutate state of your system in an undesirable way.
For lightweight system tests that are safe to run against a production environment,
please use the [CF Smoke Tests](https://github.com/cloudfoundry/cf-smoke-tests).

NOTE: Because we want to parallelize execution, tests should be written in
such a way as to be runnable individually.
This means that tests should not depend on state in other tests,
and should not modify the CF state in such a way as to impact other tests.

1. [Test Setup](#test-setup)
    1. [Install Required Dependencies](#install-required-dependencies)
    1. [Test Configuration](#test-configuration)
1. [Test Execution](#test-execution)
1. [Explanation of Test Suites](#explanation-of-test-suites)
1. [Contributing](#contributing)

## Test Setup

### Pre-Requisites for running CATS

- Install golang >= 1.6. Set up your golang development environment, per
[golang.org](http://golang.org/doc/install).
- Install preferred SCM program in order to `go get` source code.
  * [git](http://git-scm.com/)
  * [mercurial](http://mercurial.selenic.com/)
  * [bazaar](http://bazaar.canonical.com/)
- Install the [`cf CLI`](https://github.com/cloudfoundry/cli).
  Make sure that it is accessible in your `$PATH`.
- Install [curl](http://curl.haxx.se/)
- Check out a copy of `cf-acceptance-tests` and make sure that it is added to your `$GOPATH`.
  The recommended way to do this is to run:
  ```
  go get -d github.com/cloudfoundry/cf-acceptance-tests
  ```
  You will receive a warning `no buildable Go source files`;
  this can be ignored as there is no compilable go source code in the package, only test code.
- Ensure all submodules are checked out to the correct SHA.
  The easiest way to do this is by running:
  ```
  ./bin/update_submodules
  ```
- Install a running Cloud Foundry deployment to run these acceptance tests against. For example, bosh-lite.

### Updating `go` dependencies

All `go` dependencies required by CATs are vendored in the `vendor` directory.

Install [gvt](https://github.com/FiloSottile/gvt) and make sure it is available
in your $PATH. The recommended way to do this is to run:
```
go get -u github.com/FiloSottile/gvt
```

In order to update a current dependency to a specific version,
do the following:

  ```
  cd cf-acceptance-tests
  gvt delete <import_path>
  gvt fetch -revision <revision_number> <import_path>
  ```

If you'd like to add a new dependency just `gvt fetch`

### Test Configuration

You must set an environment variable `$CONFIG` which points to a JSON file that
contains several pieces of data that will be used to configure the acceptance
tests, e.g. telling the tests how to target your running Cloud Foundry deployment.

The following can be pasted into a terminal and will set up a sufficient `$CONFIG`
to run the core test suites against a
[BOSH-Lite](https://github.com/cloudfoundry/bosh-lite) deployment of CF.

```bash
cat > integration_config.json <<EOF
{
  "api": "api.bosh-lite.com",
  "admin_user": "admin",
  "admin_password": "admin",
  "apps_domain": "bosh-lite.com",
  "skip_ssl_validation": true,
  "use_http": true
}
EOF
export CONFIG=$PWD/integration_config.json
```

The full set of config parameters is explained below:

* `api` (required): Cloud Controller API endpoint.
* `admin_user` (required): Name of a user in your CF instance with admin credentials.  This admin user must have the `doppler.firehose` scope if running the `logging` firehose tests.
* `admin_password` (required): Password of the admin user above.
* `apps_domain` (required): A shared domain that tests can use to create subdomains that will route to applications also craeted in the tests.
* `skip_ssl_validation`: Set to true if using an invalid (e.g. self-signed) cert for traffic routed to your CF instance; this is generally always true for BOSH-Lite deployments of CF.
* `use_existing_user` (optional): The admin user configured above will normally be used to create a temporary user (with lesser permissions) to perform actions (such as push applications) during tests, and then delete said user after the tests have run; set this to `true` if you want to use an existing user, configured via the following properties.
* `keep_user_at_suite_end` (optional): If using an existing user (see above), set this to `true` unless you are okay having your existing user being deleted at the end. You can also set this to `true` when not using an existing user if you want to leave the temporary user around for debugging purposes after the test teardown.
* `existing_user` (optional): Name of the existing user to use.
* `existing_user_password` (optional): Password for the existing user to use.
* `persistent_app_host` (optional): [See below](#persistent-app-test-setup).
* `persistent_app_space` (optional): [See below](#persistent-app-test-setup).
* `persistent_app_org` (optional): [See below](#persistent-app-test-setup).
* `persistent_app_quota_name` (optional): [See below](#persistent-app-test-setup).
* `backend` (optional): Set to 'diego' or 'dea' to determine the backend used. If unspecified the default backend will be used.
* `include_tasks` (optional): If true, the task tests will be run. These require the task_creation feature flag to be enabled.
* `include_privileged_container_support` (optional, default false): Requires capi.nsync.diego_privileged_containers and capi.stager.diego_privileged_containers to be enabled.
* `artifacts_directory` (optional): If set, `cf` CLI trace output from test runs will be captured in files and placed in this directory. [See below](#capturing-test-output) for more.
* `default_timeout` (optional): Default time (in seconds) to wait for polling assertions that wait for asynchronous results.
* `cf_push_timeout` (optional): Default time (in seconds) to wait for `cf push` commands to succeed.
* `long_curl_timeout` (optional): Default time (in seconds) to wait for assertions that `curl` slow endpoints of test applications.
* `broker_start_timeout` (optional, only relevant for `services` suite): Time (in seconds) to wait for service broker test app to start.
* `test_password` (optional): Used to set the password for the test user. This may be needed if your CF installation has password policies.
* `timeout_scale` (optional): Used primarily to scale default timeouts for test setup and teardown actions (e.g. creating an org) as opposed to main test actions (e.g. pushing an app).
* `syslog_ip_address` (only required for `logging` suite): This must be a publically accessible IP address of your local machine, accessible by applications within your CF deployment.
* `syslog_drain_port` (only required for `logging` suite): This must be an available port on your local machine.
* `use_http` (optional): Set to true if you would like CF Acceptance Tests to use HTTP when making api and application requests. (default is HTTPS)
* `staticfile_buildpack_name` (optional) [See below](#buildpack-names).
* `java_buildpack_name` (optional) [See below](#buildpack-names).
* `ruby_buildpack_name` (optional) [See below](#buildpack-names).
* `nodejs_buildpack_name` (optional) [See below](#buildpack-names).
* `go_buildpack_name` (optional) [See below](#buildpack-names).
* `python_buildpack_name` (optional) [See below](#buildpack-names).
* `php_buildpack_name` (optional) [See below](#buildpack-names).
* `binary_buildpack_name` (optional) [See below](#buildpack-names).

#### Persistent App Test Setup
The tests in `one_push_many_restarts_test.go` operate on an app that is supposed to persist between runs of the CF Acceptance tests. If these tests are run, they will create an org, space, and quota and push the app to this space. The test config will provide default names for these entities, but to configure them, set values for `persistent_app_host`, `persistent_app_space`, `persistent_app_org`, and `persistent_app_quota_name`.

#### Buildpack Names
Many tests specify a buildpack when pushing an app, so that on diego the app staging process completes in less time. The default names for the buildpacks are as follows; if you have buildpacks with different names, you can override them by setting different names:

* `staticfile_buildpack_name: staticfile_buildpack`
* `java_buildpack_name: java_buildpack`
* `ruby_buildpack_name: ruby_buildpack`
* `nodejs_buildpack_name: nodejs_buildpack`
* `go_buildpack_name: go_buildpack`
* `python_buildpack_name: python_buildpack`
* `php_buildpack_name: php_buildpack`
* `binary_buildpack_name: binary_buildpack`

#### Route Services Test Suite Setup

The `route_services` suite pushes applications which must be able to reach the load balancer of your Cloud Foundry deployment. This requires configuring application security groups to support this. Your deployment manifest should include the following data if you are running the `route_services` suite:

```yaml
...
properties:
  ...
  cc:
    ...
    security_group_definitions:
      - name: load_balancer
        rules:
        - protocol: all
          destination: IP_OF_YOUR_LOAD_BALANCER # (e.g. 10.244.0.34 for a standard deployment of Cloud Foundry on BOSH-Lite)
    default_running_security_groups: ["load_balancer"]
```

#### Capturing Test Output
If you set a value for `artifacts_directory` in your `$CONFIG` file, then you will be able to capture `cf` trace output from failed test runs.  When a test fails, look for the node id and suite name ("*Applications*" and "*2*" in the example below) in the test output:

```bash
=== RUN TestLifecycle

Running Suite: Applications
====================================
Random Seed: 1389376383
Parallel test node 2/10. Assigned 14 of 137 specs.
```

The `cf` trace output for the tests in these specs will be found in `CF-TRACE-Applications-2.txt` in the `artifacts_directory`.

### Test Execution

There are several different test suites, and you may not wish to run all the tests in all contexts, and sometimes you may want to focus individual test suites to pinpoint a failure.  The default set of tests for the DEAs can be run via:

```bash
./bin/test_default
```

This will run the `apps`, `detect`, `internet_dependent`, `routing` and `security_groups` test suites, as well as the top level test suite that simply asserts that the installed `cf` CLI version is high enough to be compatible with the test suite.

The default tests for Diego can be run via:

```bash
./bin/diego_test_default
```

This will run the `apps`, `backend_compatibility`, `detect`, `docker`, `internet_dependent`, `routing`, `security_groups`, and `ssh` test suites, as well as the top level test suite that simply asserts that the installed `cf` CLI version is high enough to be compatible with the test suite.

For more flexibility you can run `./bin/test` and specify many more options, e.g. which suites to run, which suites to exclude (e.g. if you want to run all but one suite), whether or not to run the tests in parallel, the number of parallel nodes to use, etc.  Refer to [ginkgo documentation](http://onsi.github.io/ginkgo/) for full details.  

For example, to execute all test suites, and have tests run in parallel across four processes one would run:

```bash
./bin/test -r -nodes=4
```

Be careful with this number, as it's effectively "how many apps to push at once", as nearly every example pushes an app.

To execute the acceptance tests for a specific suite, e.g. `routing`, run the following:

```bash
bin/test routing
```

The suite names correspond to directory names.

To see verbose output from `ginkgo`, use the `-v` flag.

```bash
./bin/test routing -v
```

Most of these flags and options can also be passed to the `bin/test_default` and `bin/diego_test_default` scripts as well.

## Explanation of Test Suites

* The test suite in the top level directory of this repository simply asserts the the installed version of the `cf` CLI is compatible with the rest of the test suites.

Test Suite Name| Compatable Backend | Description
--- | --- | ---
`apps`| DEA or Diego | Tests the core functionalities of Cloud Foundry: staging, running, logging, routing, buildpacks, etc.  This suite should always pass against a sound Cloud Foundry deployment.
`backend_compatibility` | DEA and Diego are required simultaneously| Tests interoperability of droplets staged on Diego or the DEAs
`detect` | DEA or Diego | Tests the ability of the platform to detect the correct buildpack for compiling an application if no buildpack is explicitly specified.
`docker`| Diego |Test our ability to run docker containers on diego and that we handle docker metadata correctly.
`internet_dependent`| DEA or Diego | This suite tests the feature of being able to specify a buildpack via a Github URL.  As such, this depends on your Cloud Foundry application containers having access to the Internet.  You should take into account the configuration of the network into which you've deployed your Cloud Foundry, as well as any security group settings applied to application containers.
`logging`| DEA or Diego | This test exercises the syslog drain forwarding functionality. A TCP listener is deployed to Cloud Foundry. Another app is deployed to the target Cloud Foundry and bound to that listener (as a syslog drain) and the drain is checked for log messages.
`operator`| DEA or Diego |Tests in this package are only intended to be run in non-production environments.  They may not clean up after themselves and may affect global CF state.  They test some miscellaneous features; read the tests for more details.
`routing`| DEA or Diego |This package contains routing specific acceptance tests (Context path, wildcard, SSL termination, sticky sessions).
`route_services` | Diego |This package contains route services acceptance tests.
`security_groups`| DEA or Diego |This suite tests the security groups feature of Cloud Foundry that lets you apply rules-based controls to network traffic in and out of your containers.  These should pass for most recent Cloud Foundry installations.  `cf-release` versions `v200` and up should have support for most security group specs to pass.
`services`| DEA or Diego | This suite tests various features related to services, e.g. registering a service broker via the service broker API.  Some of these tests exercise special integrations, such as Single Sign-On authentication; you may wish to run some tests in this package but selectively skip others if you haven't configured the required integrations.  Consult the [ginkgo spec runner](http://onsi.github.io/ginkgo/#the-spec-runner) documention to see how to use the `--skip` and `--focus` flags.
`ssh`| Diego |This suite tests our ability to communicate with Diego apps via ssh, scp, and sftp.
`v3`| Diego| This suite contains tests for the next-generation v3 Cloud Controller API.  As of this writing, the v3 API is not officially supported.

## Contributing

This repository uses [gvt](https://github.com/FiloSottile/gvt) to manage `go` dependencies.

All `go` dependencies required by CATs are vendored in the `vendor` directory.

When making changes to the test suite that bring in additional `go` packages,
you should use the workflow described in the
[gvt documentation](https://github.com/FiloSottile/gvt#basic-usage).

### Code Conventions

There are a number of conventions we recommend developers of CF acceptance tests
adopt:

1. When pushing an app:
  * set the **backend**,
  * set the **memory** requirement, and use the suite's `DEFAULT_MEMORY_LIMIT`
unless the test specifically needs to test a different value,
  * set the **buildpack** unless the test specifically needs to test the case where
a buildpack is unspecified,
  * set the **domain**, and use the `config.AppsDomain` unless the test specifically
needs to test a different app domain.

  For example:

  ```go
  Expect(cf.Cf("push", appName,
      "--no-start"                          // don't start before setting backend
      "-b", buildpackName,                  // specify buildpack
      "-m", DEFAULT_MEMORY_LIMIT,           // specify memory limit
      "-d", config.AppsDomain,              // specify app domain
  ).Wait(DEFAULT_TIMEOUT)).To(Exit(0))

  //use the config-file specified backend when starting this app
  app_helpers.SetBackend(appName)

  Expect(cf.Cf("start", appName).Wait(CF_PUSH_TIMEOUT)).To(Exit(0))
  ```
1. Delete all resources that are created, e.g. apps, routes, quotas, etc.
This is in order to leave the system in the same state it was found in.
For example, to delete apps and their associated routes:
    ```
		Expect(cf.Cf("delete", myAppName, "-f", "-r").Wait(DEFAULT_TIMEOUT)).To(Exit(0))
    ```
1. Specifically for apps, before tearing them down, print the app guid and
recent application logs. There is a helper method `AppReport` provided in the
`app_helpers` package for this purpose.

    ```go
    AfterEach(func() {
      app_helpers.AppReport(appName, DEFAULT_TIMEOUT)
    })
    ```
1. Document the purpose of your test suite in this repo's README.md.
This is especially important when changing the explicit behavior of existing suites
or adding new suites.
1. Document all changes to the config object in this repo's README.md.
1. Document the compatible backends in this repo's README.md.
1. If you add a test that requires a new minimum `cf` CLI version, update the `cli_compatibility_test`.
1. If you add a test that is unsupported on a particular backend, add the appropriate prefix to the test description (e.g. `deaUnsupportedTag`).
