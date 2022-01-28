# Operator Manifest

[![gating status badge]][gating status link]
[![image status badge]][image status link]


A python library for processing operator manifests.

> **NOTE**: This library is being rewritten in go with CLI compatibility, see
> [operator-framework/operator-manifest-tools](https://github.com/operator-framework/operator-manifest-tools).

## Install

    $ pip install operator-manifest

> **NOTE**:  *operator-manifest* library requires that skopeo is installed on the system
## Pull Specifications

The `OperatorManifest` class can be used to identify and modify all the container image pull
specifications found in an operator
[Cluster Service Version](https://operator-framework.github.io/olm-book/docs/glossary.html#clusterserviceversion),
CSV, file. Below is a comprehensive list of the different locations within a CSV file where the
pull specifications are searched for by `OperatorManifest`:

1. RelatedImage: `spec.relatedImages[].image`
2. Annotation: any of:
   1. `metadata.annotations.containerImage`
   2. `metadata.annotations[]` for any given Object where the annotation value contains one
      or more image pull specification.
3. Container: `spec.install.spec.deployments[].spec.template.spec.containers[].image`
4. InitContainer: `spec.install.spec.deployments[].spec.template.spec.initContainers[].image`
5. RelatedImageEnv: `spec.install.spec.deployments[].spec.template.spec.containers[].env[].value`
   and `spec.install.spec.deployments[].spec.template.spec.initContainers[].env[].value` where the
   `name` of the corresponding env Object is prefixed by `RELATED_IMAGE_`.


NOTE: The Object paths listed above follow the [jq](https://stedolan.github.io/jq/manual/) syntax
for iterating through Objects and Arrays. For example, `spec.relatedImages[].image` indicates the
`image` attribute of every Object listed under the `relatedImages` Array which, in turn, is an
attribute in the `spec` Object.

When an Operator bundle image is built, the `RelatedImages` should represent the full list of
container image pull specifications. The `OperatorManifest` class provides the mechanism to
ensure this is consistently done. It's up to each build system to make the necessary calls to
`OperatorManifest`. In some cases the build system, e.g.
[OSBS](https://osbs.readthedocs.io/en/latest/), may want fully control the content of
`RelatedImages` and will purposely cause failures if `RelatedImages` is already populated.

Another functionality of the `OperatorManifest` class is the ability to modify any of the container
image pull specifications identified. This is useful for performing container registry translations
and pinning floating tags to a specific digest.

## Using the CLI

Once installed, this library provides the `operator-manifest` script which is a CLI wrapper for
using the library. Notably, there are three subcommands to the script. Use
`operator-manifest --help` for usage details.

The `pin` subcommand is a "porcelain-like" command to resolve all image references found in the
`ClusterServiceVersion` file to digests, and modify the `ClusterServiceVersion` file accordingly.

The `extract` and `replace` subcommands allow users to use a custom image reference resolver. The
first extracts all the image references from the `ClusterServiceVersion` file, while the second
modifies the image references in the `ClusterServiceVersion` file based on an input replacement
map. See `operator-manifest extract --help` and `operator-manifest replace --help` for more
details.

The `resolve` subcommand gives users direct access to the built-in resolver used by `pin`. In fact,
the `pin` subcommand is a shortcut for piping the other three subcommands together:
```
operator-manifest extract . | operator-manfiest resolve - | operator-manifest replace . -
```
Example of references (output of `extract` command):
```json
[
  "registry.fedoraproject.org/fedora:latest",
  "registry.access.redhat.com/ubi8:8.5"
]
```

Example of replacements (output of `resolve` command):
```json
{
  "registry.fedoraproject.org/fedora:latest": "registry.fedoraproject.org/fedora@sha256:6bea36502f5888df52a3dd7641af2161c4e0f35f7cbfda4e6ecd8a446b3ff5e8",
  "registry.access.redhat.com/ubi8:8.5": "registry.access.redhat.com/ubi8@sha256:060d7d6827b34949cc0fc58a50f72a5dccf00a4cc594406bdf5982f41dfe6118"
}
```

Both `replace` and `pin` will:

* Populate the `.spec.relatedImages` section of the `ClusterServiceVersion` file.
* Skip modifications to the `ClusterServiceVersion` file if the `spec.relatedImages` is already
  populated.
* Raise an error if the `spec.relatedImages` section is already populated and there are environment
  variables in the `ClusterServiceVersion` file that use the prefix `RELATED_IMAGE_`. This is
  unexpected behavior. If you have a valid use case, file an issue to remove this restriction.

## Using the Container Image

`operator-manifest` is packaged as a container image available at
`quay.io/containerbuildsystem/operator-manifest`. Using it requires volumes to be mounted on the
containers. Use `/opt/app-root/manifest-dir` for the manifest files. The working directory is by
default `/opt/app-root/workdir`. The json files created by the `pin` subcommand, for example, are
stored in that location. Don't forget to set the `:z` or `:Z` volume labels if using selinux.

Here's an example usage:
```
$ podman run --rm -it \
    -v /tmp/my-bundle-metadata:/opt/app-root/manifest-dir:Z \
    -v /tmp/my-workdir:/opt/app-root/workdir:Z \
    quay.io/containerbuildsystem/operator-manifest:latest operator-manifest pin /opt/app-root/manifest-dir \
    --authfile /opt/app-root/workdir/docker-config.json
```

## Running Tests

The testing environment is managed by [tox](https://tox.readthedocs.io/en/latest/). Simply run
`tox` and all the linting and unit tests will run.

If you'd like to run a specific unit test, you can do the following:

```bash
tox -e py37 tests/test_operator.py::TestOperatorCSV::test_valuefrom_references_not_allowed
```

### Integration Tests

Integration tests are not part of the default `tox` suite, run them with `tox -e integration`.

Tests that require authentication will, by default, be skipped. To run them:

1. Create authfile with credentials that can be used to access *registry.redhat.io*
   (this authfile is created for you when you use docker login or podman login).
2. Run `AUTHFILE_PATH=<path to authfile> tox -e integration`

Tox will execute the test command in the `tests/integration` directory. To run a specific test,
you can use the same approach as for unit tests but the path to the test file has to be relative
to `tests/integration`. Example:

```bash
tox -e integration test.py::TestReplaceCommand::test_command
```

## Dependency Management

To manage dependencies, this project uses [pip-tools](https://github.com/jazzband/pip-tools) so that
the production dependencies are pinned and the hashes of the dependencies are verified during
installation.

The unpinned dependencies are recorded in `setup.py`, and to generate the `requirements.txt` file,
run `pip-compile --generate-hashes --output-file=requirements.txt`. This is only necessary when
adding a new package. To upgrade a package, use the `-P` argument of the `pip-compile` command.

To update `requirements-test.txt`, run
`pip-compile --generate-hashes requirements-test.in -o requirements-test.txt`.

When installing the dependencies in a production environment, run
`pip install --require-hashes -r requirements.txt`. Alternatively, you may use
`pip-sync requirements.txt`, which will make sure your virtualenv only has the packages listed in
`requirements.txt`.

[gating status badge]: https://github.com/containerbuildsystem/operator-manifest/actions/workflows/gating.yaml/badge.svg?branch=master&event=push
[gating status link]: https://github.com/containerbuildsystem/operator-manifest/actions?query=event%3Apush+branch%3Amaster+workflow%3A%22gating%22
[image status badge]: https://quay.io/repository/containerbuildsystem/operator-manifest/status
[image status link]: https://quay.io/repository/containerbuildsystem/operator-manifest
