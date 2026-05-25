.. zuul:rolevar:: container_registry_credentials
   :type: dict

   This is only required for the upload and promote roles.  This is
   expected to be a Zuul Secret in dictionary form.  Each key is the
   name of a registry, and its value a dictionary with information
   about that registry.

   Example:

   .. code-block:: yaml

      container_registry_credentials:
        quay.io:
          username: foo
          password: bar

   .. zuul:rolevar:: [registry name]
      :type: dict

      Information about a registry.  The key is the registry name, and
      its value a dict as follows:

      .. zuul:rolevar:: username

         The registry username.

      .. zuul:rolevar:: password

         The registry password.

      .. zuul:rolevar:: repository

         Optional; if supplied this is a regular expression which
         restricts to what repositories the image may be uploaded.
         The registry name should be included.  The following example
         allows projects to upload images to repositories within an
         organization based on their own names::

           repository: "^quay.io/myorgname/{{ zuul.project.short_name }}.*"
