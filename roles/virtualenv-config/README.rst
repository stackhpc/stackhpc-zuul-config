Write a configuration file for virtualenv

Generally we do not require any specific virtualenv configuration.
However, there are corner cases; such as virtualenv shipping an
vendored version of setuptools that needs to be ignored or similar.

The exact action this role is taking may depend on any issues it is
working around.  See comments inline.

** Role Variables **

