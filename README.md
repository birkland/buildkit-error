# Building from cache results in corrupt images

This repo contains a `context/` directory containing a Dockerfile and some files.  The contents of the Dockerfile and files _have not changed_ since the last build and push by CI (the image is `ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`).  

The expected behavior when performing a `docker build` in the context directory is:

* When building with `--cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`, the end result should be identical to the pushed image `ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`.  (i.e. all statements in the Dockerfile will be cache hits)
* When building without a cache on a clean system, the result will be an image with new layers created for all the statements in the Dockerfile.  The resulting image will be functionally equivalent to the pushed image, but not identical (i.e. file dates and layer metadata is different)

However, the observed reality is that building from cache results in a corrupt image:

* When building with `--cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`, the build process claims that all steps are cached, ***_but the resulting image is missing content in its last layer, and therefore missing files compared to the pushed image_***
* WHen building with no cache, the image has all expected content.

## Quick start

To build a good image, do:

    docker system prune -af
    docker build --progress=plain \
      --build-arg BUILDKIT_INLINE_CACHE=1 \
      --tag local/tomcat:good context

To build a bad image, simply cache from `ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1`:

    docker system prune -af
    docker build --progress=plain \
      --cache-from ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1 \
      --build-arg BUILDKIT_INLINE_CACHE=1 \
      --tag local/tomcat:bad context

To pull the image, do:

    docker pull ghcr.io/jhu-sheridan-libraries/idc-isle-dc/tomcat:upstream-20200824-f8d1e8e-21-gd6658f1

To test an image IMAGE_TAG (e.g. `local/tomcat:bad`) for completeness, do:

     docker run --entrypoint="/bin/ls" IMAGE_TAG \ 
       /usr/local/bin/install-war-into-tomcat.sh

## The Problem

<table>
<caption>Differences in histories between bad (<code>--cache-from</code> to pull in cache) and good (no cache)</caption>
<thead>
<tr>
<td>Good (no cache) history</td>
<td>Bad (--cache-from) history</td>
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
            "created": "2021-02-04T15:19:27.9352723Z",
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