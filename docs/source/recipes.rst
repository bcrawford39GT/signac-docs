.. _recipes:

===============
Advanced How-To
===============

This is a collection of recipes on how to solve typical problems using **signac**.

.. todo::

    Move all recipes below into a 'General' section once we have added more recipes.


Migrating (changing) the project schema
==========================================

Adding/renaming/deleting keys
-----------------------------

Oftentimes, one discovers at a later stage that important :term:`parameters<parameter>` are missing from the :term:`project schema`.
For example, in the tutorial we are modeling a gas using the ideal gas law, but we might discover later that important effects are not captured using this overly simplistic model and decide to replace it with the van der Waals equation:

.. math::

   \left(p + \frac{N^2 a}{V^2}\right) \left(V - Nb \right) = N k_B T

Since the ideal gas law can be considered a special case of the equation above with :math:`a=b=0`, we could migrate all jobs with:

.. code-block:: pycon

    >>> for job in project:
    ...     job.sp.setdefault("a", 0)
    ...     job.sp.setdefault("b", 0)
    ...

The ``setdefault()`` function sets the value for :math:`a` and :math:`b` to 0 in case that they are not already present.

 * To *delete* a key use ``del job.sp['key_to_be_removed']``.
 * To *rename* a key, use ``job.sp.new_name = job.sp.pop('old_name')``.

.. note::

    The ``job.sp`` and ``job.doc`` attributes provide all basic functions  of a regular Python dict.


.. _document-wide-migration:

Applying document-wide changes
------------------------------

The safest approach to apply multiple document-wide changes is to replace the document in one operation.
Here is an example on how we could recursively replace all dot (.)-characters with the underscore-character in **all** keys [#f1]_:

.. code-block:: python

    import signac
    from collections.abc import Mapping


    def migrate(doc):
        if isinstance(doc, Mapping):
            return {k.replace(".", "_"): migrate(v) for k, v in doc.items()}
        else:
            return doc


    for job in signac.get_project():
        job.sp = migrate(job.sp)
        job.doc = migrate(job.doc)

This approach makes it also easy to compare the pre- and post-migration states before actually applying them.

.. [#f1] The use of dots in keys is deprecated. Dots will be exclusively used to denote nested keywords in the future.

Initializing state points with replica indices
==============================================

We often require multiple jobs with the same state point to collect enough information to make statistical inferences about the data. Instead of creating multiple projects to handle this, we can add a **replica_index** to the state point. For example, we can use the following code to generate 3 copies of each state point in a workspace:

.. code-block:: python

    # init.py
    import signac

    project = signac.init_project("ideal-gas-project")
    num_reps = 3

    jobs = project.find_jobs({"replica_index.$exists": False})
    for job in jobs:
        job.sp["replica_index"] = 0

    for i in range(num_reps):
        for p in range(1, 11):
            sp = {"p": p, "kT": 1.0, "N": 1000, "replica_index": i}
            project.open_job(sp).init()

Defining a grid of state point values
=====================================

Some **signac** :term:`project schemas<project schema>` are structured like a "grid" where the goal is an exhaustive search or a Cartesian product of multiple sets of :term:`parameters<parameter>`. While this can be done with nested ``for`` loops, that approach can be cumbersome for state points with many keys. Here we offer a helper function that can assist in this kind of initialization, inspired by `this StackOverflow answer <https://stackoverflow.com/a/5228294>`__:

.. code-block:: python

    # init.py
    import itertools
    import signac

    project = signac.init_project("ideal-gas-project")


    def grid(gridspec):
        """Yields the Cartesian product of a `dict` of iterables.

        The input ``gridspec`` is a dictionary whose keys correspond to
        parameter names. Each key is associated with an iterable of the
        values that parameter could take on. The result is a sequence of
        dictionaries where each dictionary has one of the unique combinations
        of the parameter values.
        """
        for values in itertools.product(*gridspec.values()):
            yield dict(zip(gridspec.keys(), values))


    statepoint_grid = {"p": range(1, 11), "kT": [1.0, 5.0, 10.0], "N": [1000, 4000]}

    for sp in grid(statepoint_grid):
        print("Initializing job", sp)
        project.open_job(sp).init()

Creating parameter-dependent operations
=======================================

Operations defined as a function as part of a **signac-flow** workflow can only have one required argument: the job.
That is to ensure reproducibility of these operations.
An operation should be a true function of the job's data without any hidden parameters.

Here we show how to define operations that are a function of one or more additional parameters without violating the above mentioned principle.
Assuming that we have an operation called *foo*, which depends on parameter *bar*, here is how we could implement multiple operations that depend on that additional parameter without code duplication:

.. code-block:: python

    class Project(FlowProject):
        pass


    def setup_foo_workflow(bar):
        # Make sure to make the operation-name a function of the parameter(s)!
        @Project.post(lambda job: bar in job.doc.get("foo", []))
        @Project.operation(f"foo-{bar}")
        def foo(job):
            job.doc.setdefault("foo", []).append(bar)


    for bar in (4, 8, 15, 16, 23, 42):
        setup_foo_workflow(bar=bar)


.. _rec_external:

Using signac-flow with MATLAB or other software without Python interface
========================================================================

The easiest way to integrate software that has no native Python interface is to implement **signac-flow** operations in combination with the ``FlowProject.operation`` ``cmd`` keyword argument.
Assuming that we have a MATLAB script called ``prog.m`` within the project root directory:

.. code-block:: matlab

    % prog.m
    function []=prog(arg1, arg2)

    display(arg1);
    display(arg2);

    exitcode = 0;

Then, we could implement a simple operation that passes it some metadata parameters like this:

.. code-block:: python

    @FlowProject.operation(cmd=True)
    def compute_volume(job):
        return "matlab -r 'prog {job.sp.foo} {job.sp.bar}' > {job.ws}/output.txt"

Executing this operation will store the output of the MATLAB script within the job's workspace within a file called ``output.txt``.

.. todo::

    Show how to use signac to initialize from the command line, or point to the signac docs for doing this.
    Clarify that in principle the only Python needed is the definition of the bash command as a string returned from a decorated Python function.


Running MPI-parallelized operations
===================================

There are basically two strategies to implement :class:`~.flow.FlowProject` operations that are MPI-parallelized, one for external programs and one for Python scripts.

MPI-operations with mpi4py or similar
-------------------------------------

Assuming that your operation is using `mpi4py`_ or similar, you do not have to change your code:

.. _mpi4py: https://mpi4py.readthedocs.io/

.. code-block:: python

    @FlowProject.operation
    def hello_mpi(job):
        from mpi4py import MPI

        print("Hello from rank", MPI.COMM_WORLD.Get_rank())

You could run this operation directly with: ``mpiexec -n 2 python project.py run -o hello_mpi``.

.. note::

    This strategy might fail in cases where you cannot ensure that the MPI communicator is initialized *within* the operation function.

.. danger::

    Read and write operations to the **job-/ and project-document** are not protected
    against race-conditions and should only be executed on one rank at a time.
    This can be ensured for example like this:

    .. code-block:: python

        from mpi4py import MPI

        comm = MPI.COMM_WORLD

        if comm.Get_rank() == 0:
            job.doc.foo = "abc"
        comm.barrier()


MPI-operations using the command line
-------------------------------------

Alternatively, you can implement an MPI-parallelized operation with the ``cmd`` keyword argument of the ``FlowProject.operation`` decorator.
This strategy lets you define the number of ranks directly within the code and is also the only possible strategy when integrating external programs without a Python interface.

Assuming that we have an MPI-parallelized program named ``my_program``, which expects an input file as its first argument and which we want to run on two ranks, we could implement the operation like this:

.. code-block:: python

    @FlowProject.operation(cmd=True, directives={"np": 2})
    def hello_mpi(job):
        return "mpiexec -n 2 mpi_program {job.ws}/input_file.txt"

The ``cmd`` keyword argument instructs **signac-flow** to interpret the operation as a command rather than a Python function.
The ``directives`` keyword argument provides additional instructions on how to execute this operation.
However, some script templates, including those designed for HPC cluster submissions, will use the value provided by the ``np`` key to compute the required compute ranks for a specific submission.

.. todo::
    Once we have templates documentation, point to it here.
    Clarify that np is just a flow convention.

.. tip::

  You do not have to *hard-code* the number of ranks, it may be a function of the job, *e.g.*: ``FlowProject.operation(directives={"np": lambda job: job.sp.system_size // 1000})``.


MPI-operations with custom script templates
-------------------------------------------

Finally, instead of modifying the operation implementation, you could use a custom script template, such as this one:

.. code-block:: bash

    {% extends base_script %}
    {% block body %}
    {% for operation in operations %}
    mpiexec -n {{ operation.directives.np }} operation.cmd
    {% endfor %}
    {% endblock %}

Storing the above template in a file called ``templates/script.sh`` within your project root directory will prepend *every* operation command with ``mpiexec`` and so on.

Forcing the execution of a specific operation for debugging
===========================================================

Sometimes it is necessary to repeatedly run a specific operation although it is not technically eligible for execution.
The easiest way to do so is to temporarily add the ``@FlowProject.post.never`` post-condition to that specific operation definition.
Like the name implies, the ``post.never`` condition is *never* true, so as long as the pre-conditions are met, that operation is *always* eligible for execution.
For example:

.. code-block:: python

    # [...]


    @Project.pre.after(bar)
    @Project.post.isfile("foo.txt")
    @Project.post.never  # TODO: Remove after debugging
    @Project.operation
    def foo(job):
        pass
        # ...

Then you could execute the operation for a hypothetical job with id *abc123*, for example with ``$ python project.py run -o foo -j abc123``, irrespective of whether the ``foo.txt`` file exists or not.

Running in containerized environments
=====================================

.. _docker: https://www.docker.com/
.. _singularity: https://sylabs.io/docs/

Using **signac-flow** in combination with container systems such as docker_ or singularity_ is easily achieved by modifying the ``executable`` *directive*.
For example, assuming that we wanted to use a singularity container named ``software.simg``, which is placed within the project root directory, we use the following directive to specify that a given operation is to be executed within then container:

.. code-block:: jinja

    @Project.operation(directives={"executable": "singularity exec software.simg python"})
    def containerized_operation(job):
        pass

If you are using the ``run`` command for execution, simply execute the whole script in the container:

.. code-block:: bash

    $ singularity exec software.simg python project.py run


.. attention::

    Many cluster environments will not allow you to **submit** jobs to the scheduler using the container image.
    This means that the actual submission, (e.g. ``python project.py submit`` or similar) will need to be executed with a **local** Python executable.

    To avoid issues with dependencies that are only available in the container image, move imports into the operation function.
    Condition functions will be executed during the submission process to determine *what* to submit, so dependencies for those must be installed into the local environment as well.

.. tip::

    You can define a decorator that can be reused like this:

    .. code-block:: python

        def on_container(func):
            return flow.directives(executable="singularity exec software.simg python")(func)


        @on_container
        @Project.operation
        def containerized_operation(job):
            pass

.. todo::

    Advanced Workflows

      1. How to do hyperparameter optimization for your awesome ML application.
      2. How to implement branched workflows.
      3. How to implement a dynamic data space (*e.g.* add jobs on-the-fly).
      4. How to implement aggregation operations.

    Parallel and Super Computing

      1. How to run and submit MPI operations.
      2. How to adjust your submit script header.
      3. How to submit a bundle of operations to a cluster.
      4. How to synchronize between two different compute environments.
      5. How to use **signac** in combination with a docker/singularity container.

Using multiple execution environments for operations
====================================================

Suppose that for a given project you wanted to run jobs on multiple
supercomputers, your laptop, and your desktop. On each of these different
machines, different operation directives may be needed. The :py:class:`FlowGroup`
class provides a mechanism to easily specify the different requirements of each
different environment.

.. code-block:: python

    # project.py
    from flow import FlowProject, directives


    class Project(FlowProject):
        pass


    supercomputer = Project.make_group(name="supercomputer")
    laptop = Project.make_group(name="laptop")
    desktop = Project.make_group(name="desktop")


    @supercomputer(
        directives={
            "ngpu": 4,
            "executable": "singularity exec --nv /path/to/container python",
        }
    )
    @laptop(directives={"ngpu": 0})
    @desktop(directives={"ngpu": 1})
    @Project.operation
    def op1(job):
        pass


    @supercomputer(
        directives=dict(nranks=40, executable="singularity exec /path/to/container python")
    )
    @laptop(directives={"nranks": 4})
    @desktop(directives={"nranks": 8})
    @Project.operation
    def op2(job):
        pass


    if __name__ == "__main__":
        Project().main()


.. tip::

   Sometimes, a machine should only run certain operations. To specify that an
   operation should only run on certain machines, only decorate the operation
   with the groups for the 'right' machine(s).

.. tip::

   To test operations with a small interactive job, a 'test' group can be used
   to ensure that the operations do not try to run on multiple cores or GPUs.

Passing command line options to operations run in a container or other environment
==================================================================================

When executing an operation in a container (e.g. Singularity or Docker) or a different environment,
the operation will not receive command line flags from the submitting process. ``FlowGroups`` can be
used to pass options to an ``exec`` command. This example shows how to use the ``run_options``
argument to tell an operation executed in a container to run in debug mode.

.. code-block:: python

    # project.py
    from flow import FlowProject


    class Project(flow.FlowProject):
        pass


    # Anything in run_options will be passed to the forked exec command when the operation is run.
    # Here we just pass the debug flag.
    debug = Project.make_group("debug", run_options="--debug")


    @debug
    @Project.post.isfile("a.txt")
    @Project.operation(directives={"executable": "/path/to/container exec python3"})
    def op1(job):
        with open(job.fn("a.txt"), "w") as fh:
            fh.write("hello world")


    if __name__ == "__main__":
        Project().main()


To run the operation with debugging, run the group called "debug" with ``python3 project.py run -o
debug``.
