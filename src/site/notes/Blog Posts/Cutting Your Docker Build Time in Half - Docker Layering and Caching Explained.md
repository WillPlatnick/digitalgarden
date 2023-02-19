---
{"dg-publish":true,"permalink":"/blog-posts/cutting-your-docker-build-time-in-half-docker-layering-and-caching-explained/"}
---


![Pasted image 20230117034817.png](/img/user/000%20LifeOS/090%20Extras/Images/Pasted%20image%2020230117034817.png)

# Introduction

In my last post, I wrote about some principles for finding and managing the right base image for your application. Today, we’re going to talk about how to optimize your Dockerfile construction by enhancing your understanding of how Docker layering works.

# Docker Layering

Before we begin, it might be helpful to talk about what a layer is, and how it works.

To get us started, let’s download the ubuntu image:

```
# docker pull ubuntu  
Using default tag: latest  
latest: Pulling from library/ubuntu  
692c352adcf2: Pull complete  
97058a342707: Pull complete  
2821b8e766f4: Pull complete  
4e643cc37772: Pull complete  
Digest: sha256:55cd38b70425947db71112eb5dddfa3aa3e3ce307754a3df2269069d2278ce47  
Status: Downloaded newer image for ubuntu:latest  
docker.io/library/ubuntu:latest
```

In the above example, we can see that the ubuntu image has 4 intermediate layers: 692c352adcf2, 97058a342707, 2821b8e766f4 and 4e643cc37772.

The way that layers work is that (almost) every instruction in your Dockerfile creates one layer. When you run a docker container, what happens behind the scenes is that these layers are laid on top of each other. So, in this example, 692c352adcf2 is the base of the foundation. After that, 97058a342707 is added on top of 692c352adcf2, and at this point, the container has everything that was in 692c352adcf2 initially, as well as all additions, deletions, and changes that 97058a342707 has in it. This process is repeated for every layer going down the chain until you have your docker container which has combined all your intermediate layers together.

# What’s the right amount of layers?

> Pro Tip #1 – Minimize the number of layers when possible, but there are penalties if you’re excessive about it.

When Docker first started, there was sometimes a fairly steep performance hit with high amounts of docker layers. Today, thanks to better filesystem drivers, it’s really not that much of a concern. However, there is still an overhead with every layer you add. 

Now, you may be thinking “well if there’s an overhead, maybe I should just try to cram as much as possible into one layer?” To some degree, this is a good idea, but there is a performance penalty you should know about. 

When you did the docker pull above, you may have noticed that docker was fetching multiple layers in parallel. If you try to shove everything you’re doing into a single layer, you won’t get as much benefit from the parallel download feature, with an end result of it taking longer to download your container.

> Pro Tip #2 – If you ever want to see the contents of each layer of your container image, check out [dive](https://web.archive.org/web/20201128040943/https://github.com/wagoodman/dive). Dive is an excellent tool for helping see what is happening at each layer of your docker image.

# Caching

Now that we’ve covered layering, let’s move into how Docker uses those layers as part of its cache.

To get started, I have setup a sample node.js repo at [https://github.com/DurinsDen/sa](https://web.archive.org/web/20201128040943/https://github.com/DurinsDen/sampleapp)[mpl](https://github.com/DurinsDen/sampleapp)[eapp](https://web.archive.org/web/20201128040943/https://github.com/DurinsDen/sampleapp) – I added a few libraries that aren’t strictly necessary to this application in order to make the npm install command take longer, in order to illustrate some points I’ll be making later.

First let’s setup a quick Dockerfile.

```
FROM node:12  
  
# Upgrade packages  
RUN apt-get update && apt-get upgrade -y  
  
WORKDIR /usr/src/app  
  
COPY . .  
  
RUN npm install  
  
  
EXPOSE 3000  
  
CMD [ "node", "app.js" ]
```

Next, let’s build this first version. I’m going to prefix it with the command ‘time’, which lets me know how long it takes to run a command.

```
# time docker build -t sampleapp .  
Sending build context to Docker daemon  3.282MB  
Step 1/7 : FROM node:12  
12: Pulling from library/node  
81fc19181915: Pull complete  
ee49ee6a23d1: Pull complete  
828510924538: Pull complete  
a8f58c4fcca0: Pull complete  
33699d7df21e: Pull complete  
923705ffa8f8: Pull complete  
c214b6cd5b8c: Pull complete  
4c73d8285dba: Pull complete  
1c58ef740d94: Pull complete  
Digest: sha256:1e17e0fdecf65b7b86e50875ad5f11ae181a8d0351806babd61b332bc32a2c15  
Status: Downloaded newer image for node:12  
 ---> 1fa6026dd8bb  
Step 2/7 : RUN apt-get update && apt-get upgrade -y  
 ---> Running in c8a0339f5661  
Ign:1 http://deb.debian.org/debian stretch InRelease  
#APT STUFF OMMITTED FOR BREVITY  
Removing intermediate container c8a0339f5661  
 ---> 3e7a3a6c64df  
Step 3/7 : WORKDIR /usr/src/app  
 ---> Running in 9a6f8e31e413  
Removing intermediate container 9a6f8e31e413  
 ---> 7fe2e982e957  
Step 4/7 : COPY . .  
 ---> 40786fceb32e  
Step 5/7 : RUN npm install  
 ---> Running in d3a14fc822a6  
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: [email protected] (node_modules/watchpack-chokidar2/node_modules/fsevents):  
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for [email protected]: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})  
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: [email protected] (node_modules/fsevents):  
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for [email protected]: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})  
  
added 484 packages from 330 contributors and audited 490 packages in 8.204s  
  
8 packages are looking for funding  
  run `npm fund` for details  
  
found 4 vulnerabilities (3 low, 1 critical)  
  run `npm audit fix` to fix them, or `npm audit` for details  
Removing intermediate container d3a14fc822a6  
 ---> 1822b28867b0  
Step 6/7 : EXPOSE 3000  
 ---> Running in 30b2e457ae73  
Removing intermediate container 30b2e457ae73  
 ---> 7e8aa1cf7bbf  
Step 7/7 : CMD [ "node", "app.js" ]  
 ---> Running in 452177a415fd  
Removing intermediate container 452177a415fd  
 ---> 130aa9966635  
Successfully built 130aa9966635  
Successfully tagged sampleapp:latest  
docker build -t sampleapp .  0.48s user 0.47s system 1% cpu 52.287 total
```

So, to do the apt-get update, apt-get upgrade, copy the files to the container and install node.js packages (npm ci –only=production), it took 52.287 seconds.

Now, let’s run the command again:

```
# time docker build -t sampleapp .  
Sending build context to Docker daemon  3.282MB  
Step 1/7 : FROM node:12  
 ---> 1fa6026dd8bb  
Step 2/7 : RUN apt-get update && apt-get upgrade -y  
 ---> Using cache  
 ---> 3e7a3a6c64df  
Step 3/7 : WORKDIR /usr/src/app  
 ---> Using cache  
 ---> 7fe2e982e957  
Step 4/7 : COPY . .  
 ---> Using cache  
 ---> 40786fceb32e  
Step 5/7 : RUN npm install  
 ---> Using cache  
 ---> 1822b28867b0  
Step 6/7 : EXPOSE 3000  
 ---> Using cache  
 ---> 7e8aa1cf7bbf  
Step 7/7 : CMD [ "node", "app.js" ]  
 ---> Using cache  
 ---> 130aa9966635  
Successfully built 130aa9966635  
Successfully tagged sampleapp:latest  
docker build -t sampleapp .  0.46s user 0.41s system 77% cpu 1.119 total
```

This time, the container build only took 1.119 seconds! How can that be?

The reason is that Docker caches intermediate layers. If the command that creates the layer doesn’t change, it’s assumed that it doesn’t need to bother running the instruction again, and uses the layer that already exists instead. See all the “Using cache” lines in the output? That’s docker not running that command from the Dockerfile.

You might be saying. “But, wait a minute…I told it to run apt-get update and apt-get upgrade in order to get the latest Debian packages, and it didn’t actually run it.” If you are thinking that, you’ve just stumbled upon a key concept.

Docker assumes that the output of your RUN command will be the same every time.

Many times, that’s a true assumption, but if you’re doing something like an apt-get upgrade or apt-get update && apt-get install -y nano, expecting to get the absolute latest version of nano, you’ll be sorely disappointed. The fix for this involves understanding a concept called cache busting.

# Cache Busting

The key to optimal Dockerfile layout is understanding what causes docker to stop using the cache. Let’s go back to the original explanation of a container image containing multiple intermediary images layered on top of each other. Because of how layering works, docker can only cache until it detects a change. Once a change is detected, the cached layers after it can’t be used, because those layers may have depended on the previous layers being in a certain state.

This leads us to…

> Pro Tip #3 – Move the things that you expect to change most-often towards the bottom of the Dockerfile.

Let’s explore this concept a little bit more. Let’s move the apt-get upgrade to the bottom of the Dockerfile, so it looks like this:

```
# time docker build -t sampleapp .  
Sending build context to Docker daemon  3.282MB  
Step 1/7 : FROM node:12  
 ---> 1fa6026dd8bb  
Step 2/7 : RUN apt-get update && apt-get upgrade -y  
 ---> Using cache  
 ---> 3e7a3a6c64df  
Step 3/7 : WORKDIR /usr/src/app  
 ---> Using cache  
 ---> 7fe2e982e957  
Step 4/7 : COPY . .  
 ---> Using cache  
 ---> 40786fceb32e  
Step 5/7 : RUN npm install  
 ---> Using cache  
 ---> 1822b28867b0  
Step 6/7 : EXPOSE 3000  
 ---> Using cache  
 ---> 7e8aa1cf7bbf  
Step 7/7 : CMD [ "node", "app.js" ]  
 ---> Using cache  
 ---> 130aa9966635  
Successfully built 130aa9966635  
Successfully tagged sampleapp:latest  
docker build -t sampleapp .  0.46s user 0.41s system 77% cpu 1.119 total
```

If we run the docker build again, you’ll notice that because we moved the apt-get update instruction later down in the file, it wasn’t able to use cached images for anything but node:12. This time, it took me 25 seconds to build.

If you build it again, you’ll see that like previously, it took about a second or two, because nothing changed.

Now, let’s just introduce a change to the app, by adding a new file called randomfile.

```
# touch randomfile  
# time docker build -t sampleapp .  
Sending build context to Docker daemon  3.282MB  
Step 1/7 : FROM node:12  
 ---> 1fa6026dd8bb  
Step 2/7 : WORKDIR /usr/src/app  
 ---> Using cache  
 ---> 4223b4b522f9  
Step 3/7 : COPY . .  
 ---> e772dde3af48  
Step 4/7 : RUN npm install  
 ---> Running in f34d347bb292  
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: [email protected] (node_modules/watchpack-chokidar2/node_modules/fsevents):  
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for [email protected]: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})  
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: [email protected] (node_modules/fsevents):  
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for [email protected]: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})  
  
added 484 packages from 330 contributors and audited 490 packages in 8.537s  
  
8 packages are looking for funding  
  run `npm fund` for details  
  
found 4 vulnerabilities (3 low, 1 critical)  
  run `npm audit fix` to fix them, or `npm audit` for details  
Removing intermediate container f34d347bb292  
 ---> 7cde9563b00d  
Step 5/7 : RUN apt-get update && apt-get upgrade -y  
 ---> Running in 1c75a9923921  
Get:1 http://security.debian.org/debian-security stretch/updates InRelease [53.0 kB]  
#APT STUFF OMMITTED FOR BREVITY  
Removing intermediate container 1c75a9923921  
 ---> 290d7562181c  
Step 6/7 : EXPOSE 3000  
 ---> Running in 6f602928eee4  
Removing intermediate container 6f602928eee4  
 ---> 6eba506c3c88  
Step 7/7 : CMD [ "node", "app.js" ]  
 ---> Running in 021e2ab84459  
Removing intermediate container 021e2ab84459  
 ---> f0f34f4b26e2  
Successfully built f0f34f4b26e2  
Successfully tagged sampleapp:latest  
docker build -t sampleapp .  0.45s user 0.39s system 3% cpu 26.027 total
```

So, we can see that docker used a cache for the WORKDIR /usr/src/app instruction, but it wasn’t able to use the cache any longer after the COPY . . line, because we added a new file. It then wasn’t able to use the cache for any subsequent layers, and the build took 26 seconds.

# Dockerfile Optimization

So, now that we understand how cache busting works, let’s use it to work in our favor.

> Pro Tip #4 – Install your packages before you copy your application files Why?

1. If you have a production application, installing npm packages for node.js, pip packages for Python, or gems for Ruby can take a LONG time due to the number of packages.
2. If you just pushed code changes up, but without changing any of your packages, then why would we want Docker to go through the effort of installing your packages again?

To demonstrate, let’s make a change to the Dockerfile to do package installation before copying the rest of my application files.

```
FROM node:12  

WORKDIR /usr/src/app  
  
COPY package.json package-lock.json ./  
  
RUN npm install  
  
COPY . .  
  
# Upgrade packages  
RUN apt-get update && apt-get upgrade -y  
  
EXPOSE 3000  
  
CMD [ "node", "app.js" ]
```

Notice how I specifically COPY package.json and package-lock.json in first and then run npm install before the COPY that brings all the files in?

After this change, anytime your application changes, and your packages don’t, you’re now going to be able to re-use the existing docker cache layer and your containers will build much faster. Let’s make a change again, and take a look at our build:

```
# time docker build -t sampleapp .  
Sending build context to Docker daemon  3.282MB  
Step 1/8 : FROM node:12  
 ---> 1fa6026dd8bb  
Step 2/8 : WORKDIR /usr/src/app  
 ---> Using cache  
 ---> 4223b4b522f9  
Step 3/8 : COPY package.json package-lock.json ./  
 ---> Using cache  
 ---> 21ef1ef68ee7  
Step 4/8 : RUN npm install  
 ---> Using cache  
 ---> 22c5ce64259f  
Step 5/8 : COPY . .  
 ---> e03fb7ec7151  
Step 6/8 : RUN apt-get update && apt-get upgrade -y  
 ---> Running in 017940d2c1b1  
Get:1 http://security.debian.org/debian-security stretch/updates InRelease [53.0 kB]  
#APT STUFF OMMITTED FOR BREVITY  
Removing intermediate container 017940d2c1b1  
 ---> 284257c93463  
Step 7/8 : EXPOSE 3000  
 ---> Running in d0765a791d87  
Removing intermediate container d0765a791d87  
 ---> ea77930ed321  
Step 8/8 : CMD [ "node", "app.js" ]  
 ---> Running in eeb475eb9212  
Removing intermediate container eeb475eb9212  
 ---> 01eb7ff2524e  
Successfully built 01eb7ff2524e  
Successfully tagged sampleapp:latest  
docker build -t sampleapp .  0.45s user 0.39s system 5% cpu 14.402 total
```

Notice that it was able to use the cache for the npm install layer. That shortened my build time from 25 seconds to 14 seconds, almost cutting my build time in half! The amount of time saved on this can be quite substantial. Every bit of optimization you can do here is crucially important to your team’s velocity, as testing often relies on docker builds, so anything that can be done to safely shorten the length of your build/test processes/pipeline is a big win for your team.

> Pro Tip #5 – Try to avoid touching system packages like apt or apk in your application container image

In the previous examples, I did an apt-get upgrade to facilitate showing you that it probably isn’t going to work the way you want. In reality, you want to avoid touching apt, yum or apk in your application containers.

I work around this issue by trying to never touch system package managers as part of my application containers, but rather, maintain a base image of some kind where I run those commands. I then use a common base image across all the other containers in my environment. Using this common base image helps keep storage space low, because all my containers can reuse the existing docker layers from my base images, rather than duplicating space in every single application.

When building my base images, I pass the –no-cache flag into the docker build command, telling Docker to never use the cache, ensuring all instructions are run every single time. I build my base images on a schedule to help keep them up to date, in a workflow I’ll blog about later. All of this helps keep the build times of your applications down, enabling devs to dev faster.