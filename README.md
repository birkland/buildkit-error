# Building from cache results in corrupt images

This repo contains a `context/` directory containing a Dockerfile and some files.  The contents of the Dockerfile and files _have not changed_ since the last build and push by CI (the image is `ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`).  

The expected behavior when performing a `docker build` in the context directory is:

* When building with `--cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`, the end result should be identical to the pushed image `ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`.  (i.e. all statements in the Dockerfile will be cache hits)
* When building without a cache on a clean system, the result will be an image with new layers created for all the statements in the Dockerfile.  The resulting image will be functionally equivalent to the pushed image, but not identical (i.e. file dates and layer metadata is different)

However, the observed reality is that building from cache results in a corrupt image:

* When building with `--cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`, the build process claims that all steps are cached, ***_but the resulting image is missing content in its last layer, and therefore missing files compared to the pushed image_***
* WHen building with no cache, the image has all expected content.

## Quick start

First, to build a corrupt image image, simply cache from `ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1` as follows:

    docker system prune -af
    docker build --progress=plain \
      --cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1 \
      --build-arg BUILDKIT_INLINE_CACHE=1 \
      --tag local/tomcat:bad context

To verify it is bad, do:

     docker run --entrypoint="/bin/ls" local/tomcat:bad /usr/local/bin/install-war-into-tomcat.sh

You should see an error:

    ls: /usr/local/bin/install-war-into-tomcat.sh: No such file or directory

Building with `--cache-from` results in a corrupt image that does not have all the expected files.

Now, pull the image we supposedly cached from:

    docker pull ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1

Test for our file on this image, and notice that it succeeds:

     docker run --entrypoint="/bin/ls" ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1 /usr/local/bin/install-war-into-tomcat.sh

Finally, clear everything from Docker and build without pulling cached layers

    docker system prune -af
    docker build --progress=plain \
      --build-arg BUILDKIT_INLINE_CACHE=1 \
      --tag local/tomcat:good context

This will take a little time to build.  If it fails due to some silly key server timeout, try again.  Now, look for our file:

    docker run --entrypoint="/bin/ls" local/tomcat:good /usr/local/bin/install-war-into-tomcat.sh

The file should be found.  Building without pulling an image for cached layers results in a good image.

## The Problem
Using our image `IMAGE="ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1"`...

Comparing the images that result from `docker pull ${IMAGE}` vs  `docker build --cache-from ${IMAGE} ...`, we observe that the `docker build --cache-from ${IMAGE}` image (which we tag as `local/tomcat:bad` above) does _not_ have the files from the last `COPY` statement in the Dockerfile.  The [build output](https://github.com/birkland/buildkit-error/blob/main/faled_build_with_cache_from.txt#L78-L98) suggests that the `COPY rootfs /` is `CACHED` (and it even looks like it is pulling in layers), but in resulting image, the layer is _empty_.  


The first few lines of `docker history local:tomcat/bad` is revealing:
```
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
b61b0abdc740   5 weeks ago    COPY rootfs / # buildkit                        0B        buildkit.dockerfile.v0
<missing>      5 weeks ago    EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      5 weeks ago    WORKDIR /opt/tomcat                             0B        buildkit.dockerfile.v0
<missing>      5 weeks ago    RUN /bin/sh -c apk-install.sh nginx &&     c…   1.15MB    buildkit.dockerfile.v0
```

As you can see, the last layer, `COPY rootfs / # buildkit` is _empty_.

Doing a pull (`docker pull ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`) reveals history that diverges in an unexplainable way:
```
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
2ea3401ac2fc   5 weeks ago    COPY rootfs / # buildkit                        7.65kB    buildkit.dockerfile.v0
<missing>      5 weeks ago    EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      5 weeks ago    WORKDIR /opt/tomcat                             0B        buildkit.dockerfile.v0
<missing>      5 weeks ago    RUN /bin/sh -c apk-install.sh nginx &&     c…   1.15MB    buildkit.dockerfile.v0
```

Notice the last COPY layer is 7.65kB in the pulled image, and 0B in the local corrupt image.

Comparing JSON metadata from these two images saved via `docker save` (pulled vs corrupt local image via --cache-from build) reveals a divergent history even stranger.  The bold lines are differances:
<table>
<thead>
<tr>
<td>Pulled image history</td>
<td>Corrupt build (--cache-from) history</td>
</tr>
</thead>
<tbody>
<tr>
<td>
<pre>
        {
            "created": "2021-02-04T15:19:27.8456647Z",
            "created_by": "WORKDIR /opt/tomcat",
            "comment": "buildkit.dockerfile.v0"
        },
        {
            <b>"created": "2021-02-04T15:19:27.9352723Z",</b>
            "created_by": "EXPOSE map[8080/tcp:{}]",
            "comment": "buildkit.dockerfile.v0",
            "empty_layer": true
        },
        {
            <b>"created": "2021-02-04T15:19:27.9352723Z",</b>
            "created_by": "COPY rootfs / # buildkit",
            "comment": "buildkit.dockerfile.v0"
        }
</pre>
</td>
<td>
<pre>
        {
            "created": "2021-02-04T15:19:27.8456647Z",
            "created_by": "WORKDIR /opt/tomcat",
            "comment": "buildkit.dockerfile.v0"
        },
        {
            <b>"created": "2021-02-04T15:19:27.8456647Z",</b>
            "created_by": "EXPOSE map[8080/tcp:{}]",
            "comment": "buildkit.dockerfile.v0",
            "empty_layer": true
        },
        {
            <b>"created": "2021-02-04T15:19:27.8456647Z",</b>
            "created_by": "COPY rootfs / # buildkit",
            "comment": "buildkit.dockerfile.v0",
            <b>"empty_layer": true</b>
        }
</pre>
</td>
</tr>
</tbody>
</table>

We actually see that history diverged from the `EXPOSE map[8080/tcp:{}]` step!  The timestamps for diverged steps differ by ~10ms.  This is utterly baffling.  Why, when building with `--cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1` does the history of the locally built image diverge from the pulled image?