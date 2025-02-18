# CEP for the cache output in v1 recipes / rattler-build

<table>
<tr><td> Title </td><td> The cache output in v1 recipes / rattler-build </td>
<tr><td> Status </td><td> In Discussion </td></tr>
<tr><td> Author(s) </td><td> Wolf Vollprecht &ltw.vollprecht@gmail.com&gt; </td></tr>
<tr><td> Created </td><td> Nov 27, 2024</td></tr>
<tr><td> Updated </td><td> </td></tr>
<tr><td> Discussion </td><td>  </td></tr>
<tr><td> Implementation </td><td> rattler-build </td></tr>
</table>

## Abstract

This CEP aims to define the cache output for v1 multi-output recipes.

## Background

Sometimes it is very useful to build some code once, and then split it into multiple build artifacts (such as shared library, header files, etc.). For this reason, `conda-build` has a special, implicit top-level build.

There are many downsides to the behavior of `conda-build`: it's very implicit, hard to understand and hard to debug (for example, if an output is defined with the same name as the top-level recipe, this output will get the same requirements attached as the top-level).

For the v1 spec we are attempting to formalize the workings of the "top-level" build. For this, we introduce a new top-level key `cache`, that has the same values as a regular output.

## Motivating Cases

This section describes specific example cases which would be used by this workflow.

### Dividing a single package variant into modular chunks

Some packages have a monolithic build system but have optional components at runtime. The most common example of this case is splitting a package into development, runtime libraries, and runtime tools; the `dav1d-feedstock` is an example of this. `dav1d-dev` contains the developer artifacts: package config files and headers. `dav1d` contains the command line utility of the same name. `libdav1d7` contains the shared libraries that other packages link to dynamically at runtime. The artifacts in these packages cannot be built separately or doing so would result in redundant work, so the package is built all together, installed to the PREFIX, then split into the output packages.

### Reusing cached build artifacts to build multiple variants

I've heard that instead of building package from scratch for each python version, some maintainers try to rebuild multiple variants of a package from a cached working directory. This might be useful if upstream hasn't designed their build system to build C and Python APIs separately.


## Specification

The top-level `cache` key looks as follows:

```yaml
cache:
  source:
    - url: https://foo.bar/source.tar.bz
      sha256: ...

  requirements:
    build:
      - ${{ compiler('c') }}
      - cmake
      - ninja
    host:
      - libzlib
      - libfoo
    # the `run` and `run_constraints` sections are not allowed here
    ignore_run_exports:
      by_name:
        - libfoo

  build:
    # only the script key is allowed here
    script: build.sh
```

<details>
  <summary>script build.sh default discussion</summary>
We had some debate wether the cache output should _also_ default to `build.sh` or should not have any default value for the `script`. This is still undecided.
</details>

When computing variants and used variables, rattler-build looks at the union of a given output and the cache. That means, even if an output does not define any requirements, the cache would still add a variant for the `c_compiler`.

When rattler-build executes the recipe, it will start by building the cache output that is appropriate for the current variant. This is computed by looking at all "used-variables" for the cache output and computing a "hash" for the cache. The build itself is executed in the same way as any other build.

The variant keys that are injected at build time is the subset used by the cache output.

When the cache build is done, the newly created files are moved outside of the `host-prefix`. Post-processing is not performed on the files beyond memoizing what files contain the `$PREFIX` (which is later replaced in binaries and text files with the actual build-prefix).

The cache restores files that were added to the prefix (conda-build also restored source files).
The cache work dir folder, including the cache sources, is also recreated at the "dirty" state of the end of the cache build. The individual outputs can add additional source files into the cached source folder.

Any new files in the prefix (from the cache) can then be used in the outputs with the `build.files` key:

```yaml
outputs:
  - package:
      name: foo-headers

    build:
      files:
        - include/**
  - package:
      name: libfoo
    build:
      files:
        - lib/**

  - package:
      name: foo-devel
    requirements:
      run:
        - ${{ pin_subpackage("libfoo") }}
        - ${{ pin_subpackage("foo-headers") }}
```

The glob list syntax can also be a dictionary with `include / exclude` keys, e.g.

```yaml
files:
  include:
    - include/**
  exclude:
    - lib/**
```

Special care must be taken for `run-exports`. For example, the `${{ compiler('c') }}` package used to build the cache is going to have run-exports that need to be present at runtime for the package. To compute run-exports for the outputs, we use the union of the requirements - so virtually, the host & build dependencies of the cache are injected into the outputs and will attach their run exports to each output.

To ignore certain run exports, the usual `ignore_run_exports` specifiers can be used in each output.

> [!NOTE]
> We have pondered other logic for attaching run exports. We could have a more complicated algorithm that attaches the run exports only to the lowest package in a chain of packages connected by `pin_subpackage(..., exact=True)`, however, duplicating the same dependencies should not really matter much to the solver.
