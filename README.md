# Switch environments before running Jupyter kernels

Sometimes, one needs to execute Jupyter kernels in a different
environment.  One could manually adjust the kernelspec files to set
environment variables or run commands before starting the kernel, but
envkernel automates this process.

In general, there are two passes: First, install the kernel, e.g.:
`envkernel lmod --name=anaconda3 anaconda3`.  This parses some options
and writes a kernelspec file with the the `--name` you specify.  When
Jupyter tries to start this kernel, it will execute the next phase.
When Jupyter tries to run the kernel, the kernelspec file will
re-execcute `envkernel` in the run mode, which does whatever is needed
to set up the environment (in this case, load the `anaconda3` module).
Then it starts the normal IPython kernel.





## Installation

```
pip install https://github.com/NordicHPC/envkernel/archive/master.zip
```

Not currently distributed through other channels, but hopefully this
will change.  This is a single-file script and can be copied just like
this.  The script must be available both when a kernel is set up, and
each time the kernel is started (and currently assumes they are in the
same location).





## General usage and common arguments

General invocation:

```
envkernel [mode] [envkernel options] [mode-specific-options]
```


General arguments usable by *all* classes during the setup phase:

These options directly map to normal Jupyter kernel install options:

* `--name $name`: Name of kernel to install (**required**).
* `--user`: Install kernel into user directory.
* `--sys-prefix`: Install to the current Python's `sys.prefix`.
* `--prefix`: same as normal kernal install option.
* `--display-name NAME`: Human-readable name.
* `--replace`: Replace existing kernel (Jupyter option, unsure what this means).
* `--language`: What language to tag this kernel.

These are envkernel-specific options:

* `--python`: Python interperter to use when invoking inside the environment (default `python`).
* `--kernel-cmd`: a string which is the kernel to start - space
  separated, no shell quoting, it will be split when saving.  The
  default is `python -m ipykernel_launcher -f {connection_file}`,
  which is suitable for IPython.  For example, to start an R kernel in
  the environment use `R --slave -e IRkernel::main() --args
  {connection_file}` as the value to this, being careful with quoting
  the spaces only once.  To find what the strings should be, copy form
  some existing kernels.  Perhaps we should make some shortcuts for
  main ones.



## Docker

Docker is currently written, but old code and needs testing and debugging before usage.  In principle, usage is quite similar to Singularity below.





## Singularity

[Singularity](https://www.sylabs.io/docs/) is a containerization
system somewhat similar to Docker, but designed for user-mode usage
without root, and with a mindset of using user software instead of
system services.


### Singularity example

```
envkernel singularity --name=NAME --contain --bind /m/jh/coursedata/:/coursedata /path/to/image.simg
```

### Singularity mode arguments

General invocation:

```
envkernel singularity --name=NAME [envkernel options] [singularity options] [image]
```

* `image`: Required positional argument: run this singularity image.

* `--pwd`: Bind-mount the current working directory and use it as the
  current working directory inside the notebook.  This may happen by
  default if you don't `--contain`.

Any unknown argument is passed directly to the `singularity exec`
call, and thus can be any normal Singularity arguments.  The most
useful Singularity options are (nothign envkernel specific here):

* `--contain` or `-c`: Don't share any filesystems by default.

* `--bind src:dest[:ro]`: Bind mount `src` from the host to `dest` in
  the container.  `:ro` is optional, and defaults to `rw`.

* `--cleanenv`: Clean all environment before executing.

* `--net` or `-n`: Run in new network namespace.  This does **NOT**
  work with Jupyter kernels, because localhost must currently be
  shared.  So don't use this unless we create proper net gateway.





## Lmod

The Lmod envkernel will load/unload
[Lmod](https://lmod.readthedocs.io/) modules before running a normal
IPython kernel.

Using envkernel is better than the naive (but functional) method of
modifying a kernel to invoke a particular Python binary, because that
will invoke the right Python interperter but not set relevant other
environment variables (so, for example, subprocesses won't be in the
right environment).

### Lmod example

This will run `module purge` and then `module load anaconda3` before
invoking an IPython kernel using the name `python`, which will
presumably be the one inside the `anaconda3` environment.

```
envkernel lmod --name=anaconda3 --purge anaconda3
```

### Lmod mode Arguments

General invocation:

```
envkernel lmod --name=NAME [envkernel options] [module ...]
```

* `module ...`: Modules to load (positional argument).  Note that if
   the module is prefixed with `-`, it is actually unloaded (this is a
   Lmod feature).

* `--purge`: Purge all modules before loading the new modules.  This
  can be safer, because sometimes users may automatically load modules
  from their `.bashrc` which will cause failures if you try to load
  conflicting ones.



## How it works

When envkernel first runs, it sets up a kernelspec that will re-invoke
envkernel when it runs.  Some options are when firs run (kernelspec
name and options), while usually most are passed through straight to
the kernelspec.  When the kernel is started, envkernel is re-invoked

Example envkernel setup command.  This makes a new Jupyter kernel
(`envkernel singularity` means singularity create mode) named
`testcourse-0.5.9` out of the image `/l/simg/0.5.9.simg` with the
Singularity options `--contain` (contain, on default mounts) and
`--bind` (bind a dir).`

```
envkernel singularity --sys-prefix --name=testcourse-0.5.9 /l/simg/0.5.9.simg --contain --bind /m/jh/coursedata/:/coursedata
```

That will create this kernelspec.  Note that most of the arguments are passed through:

```
{
    "argv": [
        "/opt/conda-nbserver-0.5.9/bin/envkernel",
        "singularity",
        "run",
        "--connection-file",
        "{connection_file}",
        "--contain",
        "--bind",
        "/m/jh/coursedata/:/coursedata",
        "/l/simg/0.5.9.simg",
        "--",
        "python",
        "-m",
        "ipykernel_launcher",
        "-f",
        "{connection_file}"
    ],
    "display_name": "Singularity with /l/simg/0.5.9.simg",
    "language": "python"
}
```

When this runs, it runs `singularity --contain --bind
/m/jh/coursedata/:/coursedata /l/simg/0.5.9.simg`.  Inside the image,
it runs `python -m ipykernel_launcher -f {connection_file}`.
envkernel parses and manipulates these arguments however is needed.





## Use with nbgrader

envkernel was orginally inspired by the need for nbgrader to securely
contain student's code while autograding.  To do this, set up a
contained kernel as above - it's up to you to figure out how to do
this properly with your chosen method (docker or singularity).  Then
autograde like such, add the `--ExecutePreprocessor.kernel_name`
option:

```
nbgrader autograde --ExecutePreprocessor.kernel_name=testcourse-0.5.9 R1_Introduction

```
