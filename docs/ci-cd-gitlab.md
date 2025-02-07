---
layout: default
title: Gitlab CI/CD example
---

# Gitlab CI/CD example

Before discussing the Gitlab CI/CD setup, let's assume you already have a working DNSControl setup. Aren't you there yet? Then first check out the '[Getting Started](https://stackexchange.github.io/dnscontrol/getting-started)' section.

## DNSControl - Demo setup

For this tutorial, there is a [Gitlab repository](https://gitlab.com/cafferata/dnscontrol/) ready with an example DNSControl setup/domain.

This is based on:
- The domain `cafferata.dev`.
- The DNS provider [TransIP](https://www.transip.nl/cp/account/api/).
- The TransIP account `cafferatax`.

For convenience, both configuration files are shown below.

- `dnsconfig.js`
  ```javascript
  var PROVIDER_NONE = NewRegistrar('none');
  var PROVIDER_TRANSIP = NewDnsProvider('transip', '-');

  D('cafferata.dev',
      PROVIDER_NONE,
      DnsProvider(PROVIDER_TRANSIP),
      DefaultTTL('1d'),
      TXT('spf', [
          'v=spf1',
          '-all'
      ].join(' '))
  );
  ```
- `creds.json`
  ```json
  {
      "transip": {
          "TYPE": "TRANSIP",
          "AccountName": "cafferatax",
          "PrivateKey": "$TRANSIP_PRIVATE_KEY"
      }
  }
  ```

## Gitlab CI - Preparation

You may have noticed that the `creds.json` file contains a variable `$TRANSIP_PRIVATE_KEY`. This variable is populated from the Gitlab CI variables and contain the TransIP API key.

```text
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDgJjUQcUrijxul
NbbN+KD3tDISPDczuJW8HYZRafOc9BgHWuYiBcE+1+9mzHi7FOujuHuTBD84vvRN
x9buaOLoJNjCvtHDtotYQMhUWF8bnNYyoIIdJGXXcRG0G0+NNreVQiZ+5IQIvhRU
PXyjvUdQoIaRc9S5NxAMbGLJUHtZ4LEdJKa7HmCXAA5dBYaMMSZjpdBJImSJAtxj
msseUkc0EiznGgsBVMOkn3VMba2gjo5RDGbfXXLUX3DTOKdwY+hdWktG/gO/3D9l
hY/gnT/MmXXko3YAcI4eQL8=
-----END PRIVATE KEY-----
```

_Example of variable `$TRANSIP_PRIVATE_KEY` contents._

<img src="{{ site.github.url }}/assets/ci-cd-gitlab/settings-ci-cd-variables.png" alt="Gitlab CI/CD variables" style="width: 900px;"/>

<img src="{{ site.github.url }}/assets/ci-cd-gitlab/settings-ci-cd-variables-insert.png" alt="Insert Gitlab CI/CD variable TRANSIP_PRIVATE_KEY" style="width: 900px;"/>

## Gitlab CI - DNSControl preview

Now it's time to apply the power of DNSControl within Gitlab CI merge requests. We'll start by adding the basic Gitlab CI setup. You can view the git diff online in the [Gitlab merge request #1](https://gitlab.com/cafferata/dnscontrol/-/merge_requests/1/diffs). The Gitlab CI setup has also been added for convenience.

`.gitlab-ci.yml`

```yaml
dnscontrol-preview:
  stage: 'test'
  image:
    name: 'stackexchange/dnscontrol'
    entrypoint: ['']
  script:
    - '/usr/local/bin/dnscontrol version'
    - '/usr/local/bin/dnscontrol check'
    - '/usr/local/bin/dnscontrol preview'
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - 'dnsconfig.js'
```

What does this YAML configuration mean?

- The `dnscontrol preview` is run within the Gitlab CI [predefined stage](https://docs.gitlab.com/ee/ci/yaml/#stages) `test` using the Docker image [stackexchange/dnscontrol](https://hub.docker.com/r/stackexchange/dnscontrol).
  - A conscious decision has been made to always use the latest version so that no maintenance is required. Of course you can choose to include a Docker image version. You do this by choosing from the [available versions](https://hub.docker.com/r/stackexchange/dnscontrol/tags), and including it in `image:` for example: `name: 'stackexchange/dnscontrol:v3.20.0'`
- Because the choice was made not to adopt a version, it's nice to know from the Gitlab CI jobs which version DNSControl is running.
  We check and validate the DNSControl set-up `dnsconfig.js`.
- Then we ask TransIP which DNS diff there is.
- **(!)** This only happens in the context of a Gitlab merge request and (_**very important**_) only when there is a change in the DNSControl configuration (`dnsconfig.js`).

Because the above Gitlab CI configuration expects a diff, we apply this by (_for example_) adding the Google Workspace SPF include.

`dnsconfig.js`

```diff
var PROVIDER_NONE = NewRegistrar('none');
var PROVIDER_TRANSIP = NewDnsProvider('transip', '-');

D('cafferata.dev',
    PROVIDER_NONE,
    DnsProvider(PROVIDER_TRANSIP),
    DefaultTTL('1d'),
    TXT('spf', [
        'v=spf1',
+       'include:_spf.google.com',
        '-all'
    ].join(' '))
);
```

From that moment everything comes together! Within the [Gitlab merge request #1](https://gitlab.com/cafferata/dnscontrol/-/merge_requests/1), a [Gitlab pipeline](https://gitlab.com/cafferata/dnscontrol/-/pipelines/656368053) with a [Gitlab job](https://gitlab.com/cafferata/dnscontrol/-/jobs/3115895010) starts running containing the command `dnscontrol preview`. The outcome of this job? The desired change that will be made within TransIP. Wow this is cool!

```shell
$ /usr/local/bin/dnscontrol version
```
```shell
dnscontrol "3.20.0" ("8bb63be8f5ed996a7ae0a21091954fcab996621b") built 26 Aug 22 14:59 UTC
```
```shell
$ /usr/local/bin/dnscontrol preview
```
```shell
******************** Domain: cafferata.dev
1 correction
#1: MODIFY TXT spf.cafferata.dev: ("v=spf1 -all" ttl=86400) -> ("v=spf1 include:_spf.google.com -all" ttl=86400)
Done. 1 corrections.
```

<img src="{{ site.github.url }}/assets/ci-cd-gitlab/ci-cd-job-output-dnscontrol-preview.png" alt="CI/CD job output for DNSControl preview" style="width: 900px;"/>

_<https://gitlab.com/cafferata/dnscontrol/-/jobs/3115895010>_

## Gitlab CI - DNSControl push

We just saw that we can view the DNSControl diff from the [Gitlab job](https://gitlab.com/cafferata/dnscontrol/-/jobs/3115895010). Now it's time to make Gitlab CI responsible for the command `dnscontrol push`.

From here several choices can be made. You can choose to have the `dnscontrol push` run as soon as a merge request is pushed to default branch (e.g. `main`), or from a Gitlab pipeline trigger within the [Gitlab web interface](https://gitlab.com/cafferata/dnscontrol/-/pipelines/new). We have opted for the [Gitlab pipeline web interface](https://gitlab.com/cafferata/dnscontrol/-/pipelines/new) so that it cannot happen that DNS changes are made from previous merge requests in default branch (e.g. `main`).

It will probably not surprise you that the basis of this Gitlab YAML configuration corresponds for 90% with the DNSControl preview. See the [Gitlab merge request #2](https://gitlab.com/cafferata/dnscontrol/-/merge_requests/2) here.

`.gitlab-ci.yml`

```yaml
dnscontrol-push:
  stage: 'deploy'
  image:
    name: 'stackexchange/dnscontrol'
    entrypoint: ['']
  script:
    - '/usr/local/bin/dnscontrol version'
    - '/usr/local/bin/dnscontrol push'
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH && $CI_PIPELINE_SOURCE == "web"'
```

What does this (new) YAML configuration mean?

- The `dnscontrol push` is run within the Gitlab CI [predefined stage](https://docs.gitlab.com/ee/ci/yaml/#stages) `deploy`.
- This only happens when you start a Gitlab pipeline from the [Gitlab web interface](https://gitlab.com/cafferata/dnscontrol/-/pipelines/new) for the default branch (e.g. `main`).

<img src="{{ site.github.url }}/assets/ci-cd-gitlab/ci-cd-pipelines-new.png" alt="Start new CI/CD pipeline from the Gitlab web interface" style="width: 900px;"/>

When we start the new [Gitlab pipeline](https://gitlab.com/cafferata/dnscontrol/-/pipelines/656368384) from the [Gitlab web interface](https://gitlab.com/cafferata/dnscontrol/-/pipelines/new), we see the Gitlab job [dnscontrol-push](https://gitlab.com/cafferata/dnscontrol/-/jobs/3115896199) which makes the changes within the DNS provider TransIP.

```shell
$ /usr/local/bin/dnscontrol version
```
```shell
dnscontrol "3.20.0" ("8bb63be8f5ed996a7ae0a21091954fcab996621b") built 26 Aug 22 14:59 UTC
```
```shell
$ /usr/local/bin/dnscontrol push
```
```shell
******************** Domain: cafferata.dev
1 correction
#1: MODIFY TXT spf.cafferata.dev: ("v=spf1 -all" ttl=86400) -> ("v=spf1 include:_spf.google.com -all" ttl=86400)
SUCCESS!
Done. 1 corrections.
```

<img src="{{ site.github.url }}/assets/ci-cd-gitlab/ci-cd-job-output-dnscontrol-push.png" alt="CI/CD job output for DNSControl push" style="width: 900px;"/>

_<https://gitlab.com/cafferata/dnscontrol/-/jobs/3115896199>_

## Gitlab CI - Duplicate YAML configuration

We have a working setup at this point that includes a `dnscontrol preview` and a `dnscontrol push` command. Well done! You might consider cleaning up the duplicate Gitlab YAML configuration. We can move the DNSControl `image` `name` and `entrypoint` to a Gitlab YAML `extends`. Then we can also move the duplicate `dnscontrol version` command to a Gitlab `before_script`. See the third (_and also last_) [Gitlab merge request #3](https://gitlab.com/cafferata/dnscontrol/-/merge_requests/3).

This eventually brings us to the following Gitlab CI setup.

`.gitlab-ci.yml`

```yaml
.dnscontrol:
  image:
    name: 'stackexchange/dnscontrol'
    entrypoint: ['']
  before_script:
    - '/usr/local/bin/dnscontrol version'

dnscontrol-preview:
  extends: '.dnscontrol'
  stage: 'test'
  script:
    - '/usr/local/bin/dnscontrol check'
    - '/usr/local/bin/dnscontrol preview'
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - 'dnsconfig.js'

dnscontrol-push:
  extends: '.dnscontrol'
  stage: 'deploy'
  script:
    - '/usr/local/bin/dnscontrol push'
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH && $CI_PIPELINE_SOURCE == "web"'
```

If you are unexpectedly unable to set up this setup, feel free to [ask questions](https://github.com/StackExchange/dnscontrol/issues/new) about it.
