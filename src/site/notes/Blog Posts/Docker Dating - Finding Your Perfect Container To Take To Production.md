---
{"dg-publish":true,"permalink":"/blog-posts/docker-dating-finding-your-perfect-container-to-take-to-production/"}
---


![Pasted image 20230117034344.png](/img/user/000%20LifeOS/090%20Extras/Images/Pasted%20image%2020230117034344.png)

# Introduction

Looking to get serious about running containers in production but aren’t sure if you’ve found the right one?

In this article, I’m going to be discussing how to pick the perfect container for you! Also, is it cool if I drop this dating metaphor? I wanted a hook to make it interesting, and I’ll be honest, I’m bored with it already.

First, we’re going to talk about choosing images on DockerHub that are safe to run in production.

Second, we’re going to talk about how to choose the right base image for your projects. In this section, we’ll go over the pros and cons of all your different options, and I’ll help you weigh the risk to the reward.

Finally, we’ll be talking about production-grade strategies to help you manage your container lifecycle.

Are you ready to get started? Let’s go!

# DockerHub

So first, let’s talk about DockerHub. When Docker first came out, it succeeded for two reasons:

First, it made container technology easy to use and accessible to everybody.

Second, it succeeded because of DockerHub, allowing people to share their images with ease.

While that ease certainly helped drive container adoption, running any random code you find on the Internet is incredibly risky. Containers make it easy not only to share your application but [also to distribute malware](https://techcrunch.com/2018/06/15/tainted-crypto-mining-containers-pulled-from-docker-hub/).

In the past, malicious users have inserted crypto-mining malware into publicly accessible DockerHub images.

While you can pretty much find anything you want on DockerHub, it doesn’t mean that what you find is necessarily safe to run.

Thankfully, DockerHub has implemented a program called Official Images to help you make better choices when it comes to downloading.

Official Images in DockerHub tend to exist for operating systems like Ubuntu and CentOS, as well as programming languages such as Java, Ruby, or Python. Official images are often maintained by the business that supports the product or official, well-regarded maintainers. In addition to being more official than your normal community image, official images are updated on a regular basis and receive vulnerability scanning as part of the publishing process.

Can a security incident still happen with a Docker Official Image? It’s possible, but far less likely. If you’d like to stay clear from DockerHub all together, Google maintains its own [base ima](https://web.archive.org/web/20201128040943/https://github.com/GoogleContainerTools/base-images-docker)[g](https://github.com/GoogleContainerTools/base-images-docker)[es](https://web.archive.org/web/20201128040943/https://github.com/GoogleContainerTools/base-images-docker) you can use.

> Pro Tip #1 – When using DockerHub, do not use anything but Docker Official Images as your base image.

You can tell if you’re using a Docker Official Image by the blue text that identifies it as a Docker Official Image directly under the name of the container.

But, the time may come where you want to run an application that doesn’t have a Docker Official Image. What do you do then?

> Pro Tip #2 – If you need to run a non-Docker Official Image, build it yourself.

For example, [https://hub.docker.com/r/mdillon/postgi](https://web.archive.org/web/20201128040943/https://hub.docker.com/r/mdillon/postgis)[s](https://hub.docker.com/r/mdillon/postgis) contains a Docker Image based off of the official PostgreSQL image that adds in an extension called PostGIS, a plugin that enables spatial and geographic objects in the PostgreSQL database. It’s a very popular image having been downloading over 10 million times, but it’s not an official image, so the best thing to do is apply some healthy skepticism as to if it’s safe.

When I want to run something like this in production, the first thing I do is pop over to the Dockerfile tab if it exists, and then click on the GitHub link.

I then take the Dockerfile, go through it line by line to make sure it isn’t doing anything crazy, and then I build it myself using trusted base images.

It’s extra work for sure, but given that there is absolutely nothing preventing someone from uploading an image that contains malicious code, it’s work that I personally deem necessary in order to maintain stable, secure systems.

If you do decide that you’re ok taking the security risk in favor of ease of use, my recommendation is to only ever consider using images that have a GitHub link and whose images are updated on a regular basis. There’s nothing stopping the takeover of that account from spreading malware, but you can at least make a more informed decision.

# Choosing the right base image for your application

Broadly speaking, I think there are 3 different options for choosing the right base image for your application.

Scratch

The first option is a special container called scratch, which is a container that contains no OS or support libraries. The ideal candidate to use with scratch is an application that can be compiled to a single binary, such as Go.

**Pros:**

1. Produces absolute smallest and most secure image (because there’s literally nothing but your binary!)

**Cons:**

1. Only specific applications can take advantage of scratch well – interpreted languages like Python and Ruby are out.
2. Because the scratch container typically only contains your binary, there’s no shell, so debugging a misbehaving application can get challenging.

## Alpine

Many popular images on DockerHub have used the Alpine distribution of Linux as a base image. Alpine Linux is a distribution that focuses on providing a minimal footprint in order to provide higher amounts of security. The philosophy is to install only what is absolutely necessary in order to run your application and nothing else.

**Pros:**

1. The smaller footprint means smaller containers and less attack surface.
2. It contains a package manager to install applications easily.

**Cons:**

1. The lack of glibc means it’s not the ideal candidate for porting many types of applications over. In addition to having issues just getting certain applications to run in general, I also ran into several issues involving DNS in Alpine containers due to the lack of glibc.

## Traditional (Debian, Ubuntu, CentOS)

You know them. You love them. They’re your favorite distributions, just packaged in container format with a few tweaks here and there. If you’ve spent any time in DockerHub, you’ll know that Debian seems to be the preferred distribution of the traditional ones.

Pros:

1. A package manager you know and love. Feels more familiar.
2. A wide array of packages available.
3. If your app runs on the OS in a VM or on bare metal, it’ll run in a container as well.

Cons:

1. More packages mean a bigger image which in turn means more possible places to attack your computer.

## My Hot Take

I tend to go towards Ubuntu because the performance hit due to size is negligible these days, and because of all the hassle I’ve experienced with Alpine. With Ubuntu, I can use a distribution with long-term support, so I don’t have to change my base image every year or so.

> Pro Tip #3: Keep your containers as small as possible, but not at the expense of having a way to troubleshoot when something goes wrong.

What makes sense to you will vary.

For instance, I tend to install more than the absolute minimum amount of packages needed for my app to work. I’ll also install bash, ping, traceroute, dig/nslookup, and curl. Having these applications installed definitely adds more potential security risks (especially curl!), but the time cost of not having those tools available when troubleshooting an issue makes it a worthy tradeoff. Instead, I mitigate the security risk I introduce by having debugging tools available in my container by using a container security product like [Sysdig Protect](https://www.sysdig.com/), [Aqua](https://www.aquasec.com/), or [TwistL](https://web.archive.org/web/20201128040943/https://www.twistlock.com/)[oc](https://www.twistlock.com/)[k](https://web.archive.org/web/20201128040943/https://www.twistlock.com/).

There is work being done in Kubernetes to introduce a concept called [ephemeral containers](https://https//kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) that will allow you to insert containers into running pods easily for troubleshooting purposes. I’m really looking forward to that because once that’s in, I’ll be able to make my images much smaller and gain some extra security points.

# Container Lifecycle in Production

Most of us understand how patching works on traditional servers: you ssh in, update some packages, maybe shift some files around, and you’re done.

Containers are a different beast. Patching becomes very different when you adopt a container infrastructure.

> Pro Tip #4 – Use the same base images everywhere you possibly can.

By using the same base images everywhere you can, you limit the number of container images that you have to check into when a security vulnerability comes out. Additionally, it also means less network traffic will be consumed when transferring your containers, lowering your bills, and decreasing application start-up time, thanks to Docker’s use of a layered file system. How far down you want to go is up to you.

## My Hot Take

In the organizations I’ve led, we decided we wanted more control of the patching lifecycle, as well as the ability to specifically control what goes into our containers (common debugging tools, a proper init system, etc…), so we maintain our own OS image, language-specific images such as Python, Ruby and Go and maybe a couple of server containers, like OpenResty. This is a gigantic undertaking, and it’s work that never stops because there are always new versions of things.

The general workflow is that the ubuntu base image build runs in our CI/CD system every single night. Once it’s built, I use a [tool from](https://web.archive.org/web/20201128040943/https://github.com/Yelp/docker-push-latest-if-changed) [Y](https://github.com/Yelp/docker-push-latest-if-changed)[elp](https://web.archive.org/web/20201128040943/https://github.com/Yelp/docker-push-latest-if-changed) that looks at the image and compares it to our latest version. If it notices a difference, it pushes the image to the registry and then notifies my CI/CD system to build new containers using the latest version.

I’ll be writing a blog article and making a video on this workflow soon.

That being said, this is a large amount of work to put into place, and for some organizations, it will be overkill. If this sounds like you, then I recommend keeping a list of approved base images and do your best to enforce their usage.

> Pro Tip #5 – Build and Deploy Often

In order for your application containers to get the latest updates from your own base images or DockerHub, they have to be rebuilt and redeployed often. Containers that are deployed and never touched are your worst nightmare when it comes to security patching. This is why having a solid CI/CD system in place before you move to containers is essential.

# Closing

I truly hope you’ve found this information useful. If you have, drop me a line in the comments below!