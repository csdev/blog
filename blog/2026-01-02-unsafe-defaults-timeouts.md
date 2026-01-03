---
title: "Unsafe Defaults: Network and Job Timeouts"
description: >-
  Explaining the common bug of missing network and job timeouts,
  including how to identify these antipatterns and how to fix them.
tags:
  - sre
---

Missing network and job timeouts are one of the most common bugs I discover
in code review. The problem arises from **unsafe defaults** &ndash; standard
settings in packages that if left unchanged, could lead to a significant
failure on production.

<!-- truncate -->

## HTTP Requests

Consider the following API call, implemented in Python using
[`requests`][requests]:

```python
import requests

# call the Github API
response = requests.get(
    "https://api.github.com/",
    headers={"Accept": "application/vnd.github+json"},
)

# raise an exception if there was an HTTP error
response.raise_for_status()
```

By default, the requests library [waits indefinitely][requests_timeout]
for the server to send its response. Normally you'll never encounter any
issues with this code. The GitHub API tends to be highly reliable.
However, in rare cases, it is possible for your network request to be
dropped or delayed in a way that does not immediately send any error
indication back to the client. In this situation, your code could hang
for a long time, even forever without raising an exception.
This bug is particularly insidious due to how infrequently it can occur,
and how it can suddenly halt your production workloads.

Fortunately the fix is simple. Specify a timeout value in seconds:

```python
response = requests.get(
    "https://api.github.com/",
    headers={"Accept": "application/vnd.github+json"},
    timeout=5,
)
```

## Shell Scripts and CI/CD

Missing request timeouts are not a language-specific problem. You may discover
unsafe defaults in many other frameworks and libraries, including shell
scripts, Dockerfiles, and CI/CD jobs. For example, did you know that cURL
has a `--max-time` setting?

```sh
curl -H 'Accept: application/vnd.github+json' \
    --max-time 5 \
    'https://api.github.com/'
```

GitHub Actions allows you to specify a [job timeout][github_timeout]
or step timeout for each part of your CI/CD workflow, which is important
if you have a complex build or test suite with many external dependencies.
GitHub's default timeout is 360 minutes, which is too high for many
use cases.

```yml
jobs:
  build:
    timeout-minutes: 10
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build
        uses: docker/build-push-action@v5
```

Similarly, Jenkins has a [timeout step][jenkins_timeout] that aborts the
wrapped code block if it takes too long to execute:

```groovy
timeout(time: 10, unit: 'MINUTES') {
    sh('npm run test')
}
```

## Database Queries

API calls are not the only source of network requests or long-running
operations in production code. Another common category is database
transactions. It is important to define timeouts for your sessions to
prevent runaway queries that consume excessive resources and impact
availability for other clients:

```sql
SET statement_timeout = '30s';
```

This example is for PostgreSQL, and will differ for other database
engines. Database tuning can be quite complicated, but at a minimum
you should be aware of which settings are available in your environment
and how to use them to protect your production app. In my experience,
I have encountered databases with unsafe default timeouts as high as
1 year.

## Infrastructure-Level Timeouts

Missing timeouts often go unnoticed because the infrastructure provides
a secondary layer of protection. For instance, if you are running a job
on AWS Lambda, the default [execution timeout][lambda_timeout] is 3 seconds.
It can be increased up to the maximum of 15 minutes, or 900 seconds:

```sh
aws lambda update-function-configuration \
    --function-name my-function \
    --timeout 900
```

In other cases, the infrastructure configuration might be insufficient
due to its own unsafe defaults. [AWS Batch][batch_timeout] has no
job timeout, though the documentation notes that Fargate tasks may lose
access to their compute resources after 14 days.

Even if you have an infrastructure-level timeout, it is a best practice to
set timeouts in your application code. This allows you to handle
timeout errors right where they occur, resulting in more meaningful
stack traces and recovery attempts. If you are unsure what timeout value
to set, it's okay to make a conservative estimate. You may be willing
to wait 30 seconds, 30 minutes, or even 30 hours for a job to complete,
but the correct answer should never be *infinity*.


[requests]: https://requests.readthedocs.io/en/stable/
[requests_timeout]: https://requests.readthedocs.io/en/stable/user/advanced/#timeouts
[github_timeout]: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idtimeout-minutes
[jenkins_timeout]: https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#timeout-enforce-time-limit
[lambda_timeout]: https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html
[batch_timeout]: https://docs.aws.amazon.com/batch/latest/userguide/job_timeouts.html
