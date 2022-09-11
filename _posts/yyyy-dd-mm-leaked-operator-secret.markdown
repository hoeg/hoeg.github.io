---
layout: post
title:  ""
date:   2021-02-07 13:33:00 +0200
categories: kubernetes operator AppSec
---

Today we will take a look at how a Kubernetes operator might leak sensitive information in a way that is not straight forward for the implementor.

using kubebuilder pattern (https://github.com/kubernetes-sigs/kubebuilder-declarative-pattern)

CRDs are defined in code and yaml 

# Enter the Humio Operator

Humio is a log management system.
They offer a SaaS solution but also the possibility to host your own Humio cluster in Kubernetes.
To manage your self hosted solution they have created a kubernetes operator to make life easier for you which is really great!
Along with setting up the cluster we also have to possibility to configure the cluster in code via the operator.

## The issue

One of the features shipped with Humio is the possibility to create alarms on certain logs.
When an alarm is triggered it invocens an action e.g. write a message to a slack channel.
This is where the issue arises.
To be able to write to a slack channel some secret is needed, either a webhook og credentials to a slack bot.
It is these secrets that are leaked due to the way the CRDs are defined.

## Diving deep

https://github.com/humio/humio-operator/blob/master/api/v1alpha1/humioaction_types.go#L91

read and assigned here https://github.com/humio/humio-operator/blob/da5e41a660339c09cf2c241f428ff83fd79990ed/controllers/humioaction_controller.go#L199

get secret using ref here https://github.com/humio/humio-operator/blob/da5e41a660339c09cf2c241f428ff83fd79990ed/controllers/humioaction_controller.go#L228

accessing kubernetes here: https://github.com/humio/humio-operator/blob/da5e41a660339c09cf2c241f428ff83fd79990ed/pkg/kubernetes/secrets.go#L86

## Fixing the issue

As described in the previous section the issue arises because the secret value is written back the the struct.
Instead of fetching the secret once and storing it on the struct we should only store the secret name and let the operator fetch the secret each time it is needed.
Having it stored on the struct leaks the secret value when the resource is descirbed through the kubernetes API.