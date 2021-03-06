---
layout: post
title:  "CircleCI and Docker Hub rate limiting - where are my secrets?"
date:   2020-12-09 12:38:00 +0200
categories: Vault CircleCI CI Docker Credentials
---
As of November 1st 2020 Docker started to roll out their [rate limiting on unauthenticated pulls from Docker Hub](https://www.docker.com/increase-rate-limits). For CircleCI users that means that it is highly advised that they always do authenticated pulls since CircleCI cloud solution uses a set pool of ip addresses for their runners.
Although CircleCI have made arrangements with Docker to not be ratelimited, this might change in the future and therefore they advice that docker pulls should be [authenticated](https://support.circleci.com/hc/en-us/articles/360050623311-Docker-Hub-rate-limiting-FAQ).

Well, easy peasy! Just add your credentials to your executors and Bob’s your uncle:
{% highlight yaml %}
executors:
  go-executor:
    docker:
      - image: circleci/golang:1.15
        auth:
          username: $DOCKERHUB_USERNAME
          password: $DOCKERHUB_PASSWORD
{% endhighlight %}

There are a couple of drawbacks doing it this way:

- The environment variables must be set for each the CircleCI project
- Password authentication to Docker Hub enables the user to do administrative tasks e.g. changing password
- In the event of a password leak, changing this password will be quite a large task if it is used for many projects

Let’s address these drawbacks and try to come up with a solution.

CircleCI supports [contexts](https://circleci.com/docs/2.0/contexts/) that can be set for workflow jobs. By adding the credentials to a context and referencing the variables in the jobs, we get rid of the hassle of manageing these on each project.

{% highlight yaml %}
workflows:
  my-workflow:
    jobs:
      - build:
          context:
            - docker-credentials
{% endhighlight %}

Now we are left with the overprivileged users. How do we handle this? Docker Hub supports [personal access token](https://www.docker.com/blog/docker-hub-new-personal-access-tokens/). These tokens cannot be used for any administrative tasks on the user itself hence they are (almost) perfect for CI usage. Additionally we can create a token for each project such that revocation is straight forward. Leveraging access tokens does mean that we cannot use the context anymore since we now have a token for each project. Alternatively we could create a single access token but this means that every project that does not need the token can still just use the context and have access to docker.

As shown there are different ways of handling this login procedure, which one you choose depends on how you want to manage your secrets.

## 3rd-party secrets- and permission manager

Centralised secrets management makes it easier to manage the lifecycle and access to your secrets. One such tool is [Hashicorp Vault](https://www.vaultproject.io/). Vault has a powerful plugin architecture that makes it possible for us to write our own secrets engine that will create a Docker Hub access token and revoke this again after we are done using this, we call these dynamic secrets since they are not hardcoded but fully managed by Vault.

Using dynamic secrets shortens the time an adversary can be able to use this secret if it is leaked, and we ensure that a job on CircleCI will always have to ask for credentials each time it needs it.

At this time it is not possible to granualte the permissions to the access tokens issued by Docker Hub. It would be nice if we could issue an access token that only had e.g. permissions pull a fixed set of images or limit the token to work on a single repository. A possible route to support this feature is by using [Open Policy Agent for Docker authorization](https://www.openpolicyagent.org/docs/latest/docker-authorization/). Combining OPA with dynamic secrets from Vault would be perfect to limit the scope of an access token and locking down the usage of these.

## Limitations in CircleCI

Unfortunately it is not possible to retrieve secrets from Vault that can be used to log into DockerHub before pulling the image for a Docker executor on CircleCI cloud solution. Until that is possible we have to store the access token In the environment for our jobs. For machine executors and remote docker executors we will be able to retrieve the secrets from Vault before we log in to Docker Hub.
