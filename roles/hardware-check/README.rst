An ansible role to check the runtime environment and fail the job if
criteria are not met. Currently only supports checking for a minimum
CPU count.

.. zuul:rolevar:: minimum_cpu_count
   :default: 2

   The minimum CPU count to consider this a valid testing environemnt
   If there are fewer CPUs an error will be raised. Note this defaults
   to 2 because you always have a least 1 and in that case wouldn't
   need an explicit check.
