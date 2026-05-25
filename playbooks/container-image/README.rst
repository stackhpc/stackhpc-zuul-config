This is one of a collection of jobs which are designed to work
together to build, upload, and promote container images in a gating
context:

  * :zuul:job:`opendev-build-container-image`: Build the images.
  * :zuul:job:`opendev-upload-container-image`: Build and stage the images in a registry.
  * :zuul:job:`opendev-promote-container-image`: Promote previously uploaded images.

The :zuul:job:`opendev-build-container-image` job is designed to be used in
a `check` pipeline and simply builds the images to verify that
the build functions.

The :zuul:job:`opendev-upload-container-image` job builds and uploads the
images to a registry, but only with a single tag corresponding to the
change ID.  This job is designed in a `gate` pipeline so that the
build produced by the gate is staged and can later be promoted to
production if the change is successful.

The :zuul:job:`opendev-promote-container-image` job is designed to be
used in a `promote` pipeline.  It requires no nodes and runs very
quickly on the Zuul executor.  It simply re-tags a previously uploaded
image for a change with whatever tags are supplied by
:zuul:jobvar:`opendev-build-container-image.container_images.tags`.
It also removes the change ID tag from the repository in the registry.
If any changes fail to merge, this cleanup will not run and those tags
will need to be deleted manually.

They all accept the same input data, principally a list of
dictionaries representing the images to build.  YAML anchors_ can be
used to supply the same data to all three jobs.

**Job Variables**

.. zuul:jobvar:: zuul_work_dir
   :default: {{ zuul.project.src_dir }}

   The project directory.  Serves as the base for
   :zuul:jobvar:`opendev-build-container-image.container_images.context`.

.. zuul:jobvar:: container_filename

   The default container filename name to use. Serves as the base for
   :zuul:jobvar:`opendev-build-container-image.container_images.container_filename`.
   This allows a global overriding of the container filename name, for
   example when building all images from different folders with
   similarily named containerfiles.

   If omitted, the default depends on the container command used.
   Typically, this is ``Dockerfile`` for ``docker`` and
   ``Containerfile`` (with a fallback on ``Dockerfile``) for
   ``podman``.

.. zuul:jobvar:: container_command
   :default: podman

   The command to use when building the image (E.g., ``docker``).

.. zuul:jobvar:: container_images
   :type: list

   A list of images to build.  Each item in the list should have:

   .. zuul:jobvar:: context

      The build context; this should be a directory underneath
      :zuul:jobvar:`opendev-build-container-image.zuul_work_dir`.

   .. zuul:jobvar:: container_filename

      The filename of the container file, present in the context
      folder, used for building the image. Provide this if you are
      using a non-standard filename for a specific image.

   .. zuul:jobvar:: registry

      The name of the target registry (E.g., ``quay.io``).  Used by
      the upload and promote roles.

   .. zuul:jobvar:: repository

      The name of the target repository in the registry for the image.
      Supply this even if the image is not going to be uploaded (it
      will be tagged with this in the local registry).  This should
      include the registry name.  E.g., ``quay.io/example/image``.

   .. zuul:jobvar:: path

      Optional: the directory that should be passed to the build
      command.  Useful for building images with a container file in
      the context directory but a source repository elsewhere.

   .. zuul:jobvar:: build_args
      :type: list

      Optional: a list of values to pass to the ``--build-arg``
      parameter.

   .. zuul:jobvar:: target

      Optional: the target for a multi-stage build.

   .. zuul:jobvar:: tags
      :type: list
      :default: ['latest']

      A list of tags to be added to the image when promoted.

   .. zuul:jobvar:: siblings
      :type: list
      :default: []

      A list of sibling projects to be copied into
      ``{{zuul_work_dir}}/.zuul-siblings``.  This can be useful to
      collect multiple projects to be installed within the same Docker
      context.  A ``-build-arg`` called ``ZUUL_SIBLINGS`` will be
      added with each sibling project.  Note that projects here must
      be listed in ``required-projects``.

.. zuul:jobvar:: container_build_extra_env
   :type: dict

   A dictionary of key value pairs to add to the container build environment.
   This may be useful to enable buildkit with docker builds for example.

.. _anchors: https://yaml.org/spec/1.2/spec.html#&%20anchor//
