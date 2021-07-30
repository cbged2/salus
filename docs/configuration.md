# Salus Configuration

Salus is designed to be highly configurable so that it can work in many different types of environments.

## Providing Configuration to Salus

Before explaining _what_ Salus configurations are possible, we will look at _how_ to provide Salus with configuration. Salus supports multiple methods of parsing config files. Firstly, there is a [default configuration](../salus-default.yaml) that Salus uses if no other configuration is provided. The defaults are geared towards failing open to prevent misconfiguration that might result in silent failures.

The simplest way to provide custom configuration to Salus, is to put a `salus.yaml` file in the root of the repository that is being scanned. This will be automatically read by Salus and any directives given will overwrite the default directives. If you have a configuration file at a different location in the repository, you can point to it with the `--config` flag (or `-c`). For instance, if you had the configuration file at the location `tests/salus-config.yaml` you can use the following command.

```sh
docker run --rm -v $(pwd):/home/repo coinbase/salus --config file://tests/salus-config.yaml
```

Salus also supports remote configuration files that can be fetched over HTTP. This is particularly useful for organization wide configurations that are centralize.

```sh
docker run --rm -v $(pwd):/home/repo coinbase/salus --config https://salus-config.internal.net/salus.yaml
```

A third method of providing configuration URIs is via the environment variable `SALUS_CONFIGURATION`. If this envar is set, it will take precendence over the flag URI.

### Cascading Configurations

When using a global configuration provided over HTTP, you might still want to modify Salus for special cases relevant only to the repository being scanned. To do this, you can append multiple configuration files by listing the configuration URIs in order of ascending precedence. The hash represented by the YAML is deep merged with the next one. This means that keys are preserved but values are overwritten unless the value is also a Hash, in which case the algorithm recurses. A space (`" "`) is used to delimit each URI.

```sh
docker run             \
  --rm                 \
  -v $(pwd):/home/repo \
  coinbase/salus --config "https://salus-config.internal.net/global.yaml file://local-salus-config.yaml"
```

## Salus Configurations

Each configuration file should be valid YAML and may contain the following values.

```yaml
---
# String
# Used in the report to identify the project being scanned.
project_name: "my-repo"

# String
# Used in the report for any additional information
# that might be used by the consumer of the Salus report.
custom_info: "PR-123"

# Numeric (Integer or Float)
# Defines how many seconds Salus should run its scans before timing
# out (defaults to indefinite amount of time if not specified)
salus_timeout_s: 120

# Array[Hash[String=>String]]
# Defines where to send Salus reports and in what format.
#
# Each Hash must contain keys for `uri` and `format`.
# URIs can point to either the local file system or remote HTTP destinations.
# - Request paramters (optional) can be included for HTTP destinations with the `params` field
#   - if the report parameter is included, the report paramater would contain the salus report
#   - when `report` is not included in params, the salus report will be located in the body of the request sent
# The available formats are `json`, `yaml`, `txt` and `sarif`.
# `verbose` is an optional key and defaults to false.
# 
# Each report hash can add post paramaters using the `post` key , 
# - Salus reports can be sent as a report paramater by specifying the paramater name in `salus_report_param_name`
# - additional post paramaters can be specified through the `additional_params` field
#
# Additional options are also available for sarif using the optional keyword: sarif_options
# The available options for the sarif_options keyword are:
# 1) `include_suppressed: true/false` -This option allows users to include/exclude suppressed/excluded results 
#    in their sarif reports. Currently this is supported for NPM audit reports
reports:
  - uri: file://tests/salus-report.txt
    format: txt
  - uri: https://salus-config.internal.net/salus-report
    format: json
    verbose: true
  - uri: https://salus-config.internal2.net/salus-report
    format: json
    verbose: true
    post:
      salus_report_param_name: 'report'
      additional_params:
        repo: 'Random Repo'
        user: 'John Doe' 
  - uri: file://tests/salus-report.sarif
    format: sarif
  - uri: file://tests/salus-report.sarif
    format: sarif
    sarif_options:
      include_suppressed: true

# Hash with build info. Can contain arbitrary keys.
builds:
  url: "{{BUILD_URL}}"   # See Envar Interpolation section
  service_name: circle_ci

# Array[String] or String.
# Array[String] - lists all the scanner to execute if Salus determines that
#                 they are relevant to the source code in the repository.
# String        - value of "all" or "none" which will use all defined scanners or none of them respectively.
active_scanners:
  - PatternSearch
  - Brakeman
  - BundleAudit
  - NPMAudit

# Array[String] or String.
# Array[String] - lists all scanners that should cause Salus to exit with
#                 a non-zero status if they find a security vulnerability.
#                 This is particularly useful for failing CI builds.
# String        - value of "all" or "none" which will use all defined scanners or none of them respectively.
enforced_scanners:
  - PatternSearch
  - Brakeman

# Hash[String=>Hash]
# Defines configuration relevant to specific scanners.
scanner_configs:
  BundleAudit:
    ignore:
      - CVE-XXXX-YYYY # irrelevant CVE which does not have a patch yet
```

Special configuration that exist for particular scanners is defined in the [scanners directory](/docs/scanners).

## Envar Interpolation

It's sometimes useful, especially in CI clusters, to have a generalized configuration file that can reference environment variables. Salus will interpolate the configuration files before parsing it. To reference an environment variable, put the name of the envar in two parenthesis, `{{ENVAR_NAME}}`.

```yaml
project_name: "{{GITHUB_ORG}}-{{GITHUB_REPO}}-{{COMMIT_SHA}}"
```
