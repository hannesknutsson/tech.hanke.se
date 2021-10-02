+++
author = "Hannes Knutsson"
title = "Issuer Hell"
date = 2021-10-02T13:01:49+02:00
tags = [
  "certificate",
  "issuer",
  "kubernetes",
  "letsencrypt"
]
draft = false
+++

# Self referencing meta post

This whole post is about setting up the infrastructure around this site.

While doing so I was wrestling the certificate issuing process for quite some time. But to arrive at the problem, and eventually the solution, we have to go back a bit.

## The building blocks

Before you can serve a site correctly in Kubernetes there are a set of resources that have to be set up correctly. I am talking about the [ingress controller](https://docs.nginx.com/nginx-ingress-controller/) and the [cert manager](https://cert-manager.io/docs/)

I will include a mini explaination on both below. Skip them if you are aware of the concepts.

##### Ingress Controller

An ingress controller is like a router for web traffic that arrives at your cluster on a specific port. Basically it is a web server that proxies incoming traffic to other services in the cluster based on the host header in the requests.

##### Ingress

Ingress rules in Kubernetes is how you define how to open up a service to clients outside of the cluster through the ingress controller. Ingress rules contain the host name that the service should be associated with, what certificate issuer the ingress controller should use to get a certificate for this hostname, and to what service the traffic should be forwarded to. A lot more can be done in an ingress rule, but these are the main parameters most ingress rules have.

##### Cert Manager

The cert manager provides the ingress controller with certificates based on the requested hostname and issuer defined in the ingress rules.

## Setting up the basics

This was the first time I ever set up this sort of infrastructure. I was excited, but that is not much help when there are a thousand things to do wrong along the way.

Using [Helm](https://helm.sh/) charts in [ArgoCD](https://argoproj.github.io/cd), I tried to set up the ingress controller and cert manager until everything showed green lights. Eventually it all seemed to be working, but I had no way of testing it. Yet.

I set up a simple static nginx web server with some html files in with the goal to access it over https with valid letsencrypt signed certificates. After some fiddling I managed to get it working by referencing the issuer object in the ingress annotations. I got this test app working relatively quickly and my self esteem was high. Everything worked.

## Getting cokcy

After my great success, I wanted to get as many services going as soon as possible. Life was good. I set up some more static sites and the [Minecraft](https://mcmap.hanke.se/?worldname=world&mapname=surface-SW&zoom=3&x=195&y=64&z=1296) server. Everything was still working great!

It was time to clean up. I noticed that my ```kubectl get pods``` commands were returning more and more entries as I moved on, and it was becoming increasingly difficult to quickly find what I was looking for. It was time for namespaces.

I divided and moved all applications to their own namespaces so that even running ```kubectl get all -n <some namespace>``` wasn't returning to much information. All was running like a charm.

Then I came to think of creating this blog. And I started setting it up.

## Well, well, well, how the turntables

This is the point where everything turned into an infuriating dumpsterfire of debugging with no prior experience or competence within the domain at all.

My new site was not able to get its new certificate signed.

I had set it up exactly like I had set up the other static sites I was currently serving with no problem. For some reason the ingress for this web app was just not able to resolve the issuers I had referenced, so it didn't have any way to get the new certificate signed.

#### Segregation

After some debugging and asking around in a public Discord server I found that for the ingress to be able to resolve an issuer object, they must be located within the same namespace. This made sense since I had had all the other services in the same namespaces as the issuers when I set them up, allowing them to find the issuers and get the certificates signed. But now that I had split everything into namespaces no new certificates could be signed since the issuers could not be resolved.

I solved this by creating an issuer in each namespace that required signing of certificates. Bam. It was working again. The new site test.hanke.se was (a)live and kickin'.

#### Staying [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)

The solution to the namespace problem left me with a foul taste in my fingertips. I had multiple copies of the issuer objects (staging issuer/prod issuer). One of each in every namespace. Surely this could not be the way it was meant to be done. Repeating yourself like this is how stuff get messy and uncontrollable. Back to the intrnet to look for advice.

After some time I realized there are different types of issuers one can use in Kubernetes. Issuer objects (that I had used) can only be resolved from the namespace they are in themselves. ClusterIssuers on the other hand are reachable from any and all namespaces in the cluster. I promptly replaced all my issuers with two cluster issuers. One for staging and one for prod.

#### Syntax schmyntax

With new found hope and a now moderate level of self confidence I started setting up tech.hanke.se, but instantly ran into a new wall. All sites I served were doing well, except for the new tech.hanke.se. The ingress could still not find the issuer I had configured it to use.

I checked whether the working sites could still get new certificates signed by deleting the certificate resource from the ingresses. Yep, they very quickly managed to recover to a working state with correct certificates. Weird... They were configured exactly the same as the broken tech blog service, but for some reason they were fine.

At this point I spent a *LOT* of time figuring out what was wrong. The new blog site seemed to have entered some sort of isolated state where it couldn't reach my cluster issuers no matter what. But after some time I figured it out.

Initially I had configured the ingresses issuer using the annotation:

```cert-manager.io/issuer: "letsencrypt-staging"```

This is correct. So when changing to using cluster issuers, it came naturally to me to change the annotation to:

```cert-manager.io/clusterissuer: "letsencrypt-staging"```

This turned out to have been a mistake. The correct syntax was:

```cert-manager.io/cluster-issuer: "letsencrypt-staging"```

With a hyphen in the middle. After adding the hyphen, the certificate was signed successfully.

But how were all the other sites working so well despite having the same error in their ingress configurations?

After messing around a bit I found that the reason these services were unphased was because before they generate new certificates and issue new certificate requests, they cleverly check for secrets matching the requested hostname. If there is already one, the certificate can be re-used and taken from there. So had I only deleted the secret resource in addition to deleting the certificate these services would fall into the same state of not being able to resolve the cluster issuers as well.

#### Always in such a hurry

Now I was eager to test out the fruit of my hard earned work. I headed to tech.hanke.se and instantly got stopped by my browser. The certificate was not legitimate. How? Everything was looking good in ArgoCD.

When I after a while inspected the certificate I realized the issuer I used was letsencrypts staging issuer, so the certificate I had received was not from their real CA.

After switching to my prod setup it still didn't work since I had just duplicated my staging cluster issuer, not taking into account that the prod environment and the staging environment have different URL:s for the ACME servers, so when I thought I was getting the prod version I was still just issuing staging certificates.

After fixing this final error everything was good.

## Summary

Trying the same thing over and over again is a great method for accomplishing nothing. Had I read more carefully I might have noticed the syntax differences and not spent so much time doing stupid stuff. But you know what they say.

> A few hours of trial and error can save you several minutes of reading documentation

Amen.
