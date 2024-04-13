---
layout: post
show_meta: true
title: Using Docker Buildx cache
header: Using Docker Buildx cache
date: 2024-04-12 00:00:00
summary: Using Docker Buildx cache to improve docker build times
categories: docker github github-actions cache
author: Chee Yeo
---

[Docker Cache Storage]: https://docs.docker.com/build/cache/backends/
[Inline Cache]: https://docs.docker.com/build/cache/backends/inline/
[Local Cache]: https://docs.docker.com/build/cache/backends/local/
[Github cache Local Cache]: https://docs.docker.com/build/ci/github-actions/cache/#local-cache
[Github Actions Cache]: https://docs.docker.com/build/cache/backends/gha/
[Registry Cache]: https://docs.docker.com/build/cache/backends/registry/


In a recent docker build on Github Action, the build time has increased to over an hour due to the complexity of the build process. To improve build times, I started to investigate the use of [Docker Cache Storage] in the docker build process.

There are four main types of cache storage:

* Inline
* Local
* Github Actions
* Registry


The other two cache storage involve using S3 on AWS and blob storage on Azure, which is outside the scope of this discussion.

[Inline Cache] stores or embeds the cache inside the image and its the default cache mode. It only supports the **min** cache mode. It is not scalable for multi-stage builds as the cache is not persisted externally in a separate location for reuse. There is also no separation between the artifacts and cache files.

[Local Cache] allows you to store the cache files in a separate directory on disk. Docker stores a digest of the build in OCI image layout in the target cache directory. However, the previous caches remain indefintely since the caches are addressed via digests and will result in exponential growth of the cache directory, which will require manual maintenance. In [Github cache Local Cache], we can leverage the **actions/cache** and **local cache exporter** to use it but it requires additional steps to workaround the growing cache size issue. In my own usage, this has often resulted in failed builds for complex workflows due to the cache size issue.

[Github Actions Cache] is a familiar cache backend when running docker buildx in github actions. However, it stores the cache files in the provided Github Cache storage, which has a maximum default of **10 GB**. Since each of the job in my workflow averages around 7 GB, when using this cache backend, it resulted in jobs being run twice as the cache was overwritten and could not be found. This further lengthens the build time.

[Registry Cache] allows you to separate the cache files from the build artifacts. The final build image can be exported without the cache files, compared with using the inline cache. It supports multi-stage builds by caching the different build stages rather than the final stage only. You can also pass advanced options such as compression and cache mode to use.

Below is a simplified example of using the [Registry Cache] in my workflow:
{% highlight yaml %}
-   name: Build Image
    uses: docker/build-push-action@v5
    with:
        file: Dockerfile.custom
        load: true
        push: false
        tags: chee/app:latest
        cache-from: type=registry,ref=chee/app-cache:latest
        cache-to: type=registry,ref=chee/app-cache:latest,mode=max

-   name: Run tests
    run: |
        echo "Run tests on built image"

-   name: Push Image
    uses: docker/build-push-action@v5
    with:
        file: Dockerfile.custom
        push: true
        tags: chee/app:latest
        cache-from: type=registry,ref=chee/app-cache:latest
        cache-to: type=registry,ref=chee/app-cache:latest,mode=max
{% endhighlight %}

Authentication to docker hub is already done in previous steps. The cache repository is set to private. The cache is stored in a remote docker hub repository of **chee/app-cache** with a tag of **latest** to match the image tag. The cache mode is set to **max** to cache all the intermediate steps of a multi-stage build.

The first step builds the image but only stores it on the runner. This is to enable the intermediate steps to be run such as running further tests on the build image or security scans such as docker scout. When it reaches the final push image step, it fetches the cache files from the remote registry and since it exists, it will push the image directly to docker hub without rebuilding the image again.

By adopting this approach, I managed to cut the build time down to 40 mins compared to the previous 1.5 hours. In addition, since I'm using self-hosted runners, any of the credentials and secrets are not cached in the Github Cache storage externally thereby improving security.

To utilize registry cache mode locally, you would need to create a new buildx instance using a different driver apart from the default **docker** driver since its not supported.

For example:
{% highlight shell %}
docker buildx create \
  --name=container \
  --driver=docker-container \
  --use --bootstrap container

docker buildx build --push -t <registry>/<image> --builder=container \
  --cache-to type=registry,ref=<registry>/<cache-image>,mode=max \
  --cache-from type=registry,ref=<registry>/<cache-image> .
{% endhighlight %}

To summarize, if you are running multi-stage or complex docker builds on github actions, consider using the [Registry Cache] as it is highly configurable and efficient, compared to the other cache backends.

Hope this helps someone.

H4PPY H4CK1NG !
