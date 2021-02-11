---
layout: post
title:  "From static to dynamic with Vault + DockerHub"
date:   2021-02-07 13:33:00 +0200
categories: Vault DockerHub Credentials DevSecOps
---

Often in CI pipelines we need to authenticate to a docker image repository. Docker Hub is widely used and easily available but using you username and password to authenticate comes with some caveats. Your credentials can be used to administrative activitetes like chaning your password or email on your account. Actions that you would never do as part of your CI. It is also hard to manage your credentials if they are copied around into different build jobs and a leakage of your password in one place results in resetting the password and chaning it everywhere.

## DockerHub access tokens

DockerHub supports [access tokens](https://docs.docker.com/docker-hub/access-tokens/) which fits better into the CI and other integreations. While you can issue an access token for each build job you still have to manage these credentials by copying them and revoking in case of a leaks. Using access tokens provide a lot security upsides compared to using your username and password. But what if we did not have manage the lifecycle of these tokens our selves? We would get the benifites of access tokens without having to do any administrative tasks.

## Vault plugin to the rescue

To make this possible I created a [secrets plugin for Hashicorp Vault](https://github.com/hoeg/vault-plugin-secrets-dockerhub). Note that the code is currently just written as a proof of concept, but I hope to clean it up and make it more robust in the future.

When the plugin has been registered you can configure the it to create temporary access tokens managed by Vault.

First we write a configuration:

```
$ vault write dockerhub/config/hoeg namespace=hoeg,my-organization password=...
Success! Data written to: dockerhub/config/hoeg
```

You can specify multiple namespaces by writing them as a commaseparated list.

Afterward we can inspect our configuration by calling

```
$ vault read dockerhub/config/hoeg
Key          Value
---          -----
namespace    [hoeg my-organization]
ttl          5m0s
username     hoeg
```

The DockerHub plugin is now installed and configured. We now need to allow Vault users to issue access tokens. We can do this by adding a policy to our user: 

{% highlight hcl %}
path "dockerhub/token/hoeg/*" {
    capabilities = [
        "write"
    ]
}
{% endhighlight %}

or if we only want to be able to access a the `my-organization` namespace we add the following policy:

{% highlight hcl %}
path "dockerhub/token/hoeg/my-organization" {
    capabilities = [
        "write"
    ]
}
{% endhighlight %}

this illustrates that we can tighten the scope even more using access tokens this way. Unfortunately Docker Hub does not support scoping wrt. usage of the token. I would be really nice if we could ask for read only or push only token. Perhaps that will become available in the future.

To get your temporary Docker Hub credentials just call:

```
$ vault write dockerhub/token/hoeg/my-organization label=release-build
Key                Value
---                -----
lease_id           dockerhub/token/hoeg/epadctf/aYGz8UagEFryAsuFGd9s75xE
lease_duration     5m
lease_renewable    false
namespace          my-organization
token              81337d8a-789c-4767-b603-652fef582903
username           hoeg
uuid               6d4c51ef-bb1e-45da-898d-f08f55a50c2b
```

we get the token and see that the token will be revoked after 5 minutes by Vault.

## Closing

Great! By doing this as part of our pipeline we can get fresh credentials when we need them and should they leak we have minimized the window in which they are usable. The plugin is written in about a days time which shows that it is quite easy to go from static secrets to dynaminc using Vaults secrets plugin. I hope that this project will serve as a help for others who might see a need for at plugin for some other usecase as I found the documentation and articles provided by Hashicorp a bit lacking. I will do a writeup about the problems I encountered while writing this plugin at a later time. 
