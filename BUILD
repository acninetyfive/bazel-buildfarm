load("@buildifier_prebuilt//:rules.bzl", "buildifier")
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_image_index", "oci_push", "oci_tarball")
load("@rules_pkg//:pkg.bzl", "pkg_tar")
load("@rules_pkg//pkg:mappings.bzl", "pkg_attributes", "pkg_files")
load("//:jvm_flags.bzl", "server_jvm_flags", "worker_jvm_flags")
load("//container:defs.bzl", "oci_image_env")

package(default_visibility = ["//visibility:public"])

# Made available for formatting
buildifier(
    name = "buildifier",
)

# == Docker Image Creation ==
# When deploying buildfarm, you may want to include additional dependencies within your deployment.
# These dependencies can enable features related to the observability and runtime of the system.
# For example, "debgging tools", "introspection tools", and "exeution wrappers" are examples of dependencies
# that many need included within deployed containers.  This BUILD file creates docker images that bundle
# additional dependencies alongside the buildfarm agents.
ARCH = [
    # keep sorted
    # "aarch64", # TODO
    "amd64",
]

DEFAULT_IMAGE_LABELS = {
    "org.opencontainers.image.source": "https://github.com/bazelbuild/bazel-buildfarm",
}

DEFAULT_PACKAGE_DIR = "app/build_buildfarm"

# == Execution Wrappers ==
# Execution wrappers are programs that buildfarm chooses to use when running REAPI actions.  They are used for
# both sandboxing, as well as changing runtime behavior of actions.  Buildfarm workers can be configured
# to use execution wrappers directly through a configuration called "execution policy".  Execution wrappers
# can be stacked (i.e. actions can run under multiple wrappers).  Buildfarm may also choose different
# execution wrappers dynamically based on exec_properties.  In order to have them available to the worker, they should
# be provided to a java_image as a "runtime_dep".  Buildfarm workers will warn about any missing execution wrappers
# during startup and what features are unavailable due to their absence.

# == Execution Wrapper Compatibility ==
# "process-wrapper" and "linux-sandbox" are sourced directly from bazel.  Users may want to ensure that the same
# bazel version is used in buildfarm agents as is used by bazel clients.  There has not been any known issues due
# to version mismatch, but we state the possibility here.  Some execution wrappers will not be compatible with all
# operating systems.  We make a best effort and ensure they all work in the below images.
pkg_tar(
    name = "execution_wrappers",
    srcs = [
        ":exec-wrapper-files",
        ":exec-wrapper-helpers",
    ],
    package_dir = DEFAULT_PACKAGE_DIR,
    tags = ["container"],
)

pkg_tar(
    name = "telemetry_tools",
    srcs = [
        ":opentelemetry-javaagent",
    ],
    package_dir = DEFAULT_PACKAGE_DIR,
    tags = ["container"],
)

pkg_files(
    name = "tini.binary",
    srcs = ["@tini//file"],
    attributes = pkg_attributes(
        mode = "0555",
    ),
    renames = {
        "@tini//file": "tini",
    },
    tags = ["container"],
)

pkg_files(
    name = "opentelemetry-javaagent",
    srcs = ["@opentelemetry//jar"],
    attributes = pkg_attributes(
        mode = "0444",
    ),
    renames = {
        "@opentelemetry//jar": "opentelemetry-javaagent.jar",
    },
    tags = ["container"],
)

cc_binary(
    name = "as-nobody",
    srcs = select({
        "//config:windows": ["as-nobody-windows.c"],
        "//conditions:default": ["as-nobody.c"],
    }),
    tags = ["container"],
)

pkg_files(
    name = "exec-wrapper-files",
    srcs = [
        ":as-nobody",
        "@bazel//src/main/tools:linux-sandbox",
        "@bazel//src/main/tools:process-wrapper",
        "@skip_sleep",
        # The delay wrapper is only intended to be used with the "skip_sleep" wrapper.
        "delay.sh",
        "macos-wrapper.sh",
    ],
    attributes = pkg_attributes(
        mode = "0555",
    ),
    tags = ["container"],
)

pkg_files(
    name = "exec-wrapper-helpers",
    srcs = ["@skip_sleep//:skip_sleep_preload"],
    attributes = pkg_attributes(
        mode = "0444",
    ),
    prefix = DEFAULT_PACKAGE_DIR,
    renames = {
        "@skip_sleep//:skip_sleep_preload": "skip_sleep_preload.so",
    },
    tags = ["container"],
)

pkg_tar(
    name = "layer_tini_amd64",
    srcs = [":tini.binary"],
    tags = ["container"],
)

pkg_tar(
    name = "layer_buildfarm_server",
    srcs = ["//src/main/java/build/buildfarm:buildfarm-server_deploy.jar"],
    package_dir = DEFAULT_PACKAGE_DIR,
    tags = ["container"],
)

pkg_tar(
    name = "layer_buildfarm_worker",
    srcs = ["//src/main/java/build/buildfarm:buildfarm-shard-worker_deploy.jar"],
    package_dir = DEFAULT_PACKAGE_DIR,
    tags = ["container"],
)

pkg_tar(
    name = "layer_minimal_config",
    srcs = ["@build_buildfarm//examples:example_configs"],
    package_dir = DEFAULT_PACKAGE_DIR,
    tags = ["container"],
)

pkg_tar(
    name = "layer_logging_config",
    srcs = ["@build_buildfarm//src/main/java/build/buildfarm:configs"],
    package_dir = DEFAULT_PACKAGE_DIR + "/src/main/java/build/buildfarm",
    tags = ["container"],
)

oci_image_env(
    name = "env_server",
    configpath = "/" + DEFAULT_PACKAGE_DIR + "/config.minimal.yml",
    jvm_args = server_jvm_flags(),
)

oci_image(
    name = "buildfarm-server_linux_amd64",
    base = "@amazon_corretto_java_image_base",
    entrypoint = [
        "java",
        "-jar",
        "/" + DEFAULT_PACKAGE_DIR + "/buildfarm-server_deploy.jar",
    ],
    env = ":env_server",
    labels = DEFAULT_IMAGE_LABELS,
    tags = ["container"],
    tars = [
        # do not sort
        ":layer_logging_config",
        ":layer_minimal_config",
        ":telemetry_tools",
        ":layer_buildfarm_server",
    ],
)

oci_image_env(
    name = "env_worker",
    configpath = "/" + DEFAULT_PACKAGE_DIR + "/config.minimal.yml",
    jvm_args = worker_jvm_flags(),
)

oci_image(
    name = "buildfarm-worker_linux_amd64",
    base = "@ubuntu_lunar",
    entrypoint = [
        # do not sort
        "/tini",
        "--",
        "java",
        "-jar",
        "/" + DEFAULT_PACKAGE_DIR + "/buildfarm-shard-worker_deploy.jar",
    ],
    env = ":env_worker",
    labels = DEFAULT_IMAGE_LABELS,
    tags = ["container"],
    tars = [
        # do not sort
        ":layer_tini_amd64",
        ":layer_logging_config",
        ":layer_minimal_config",
        ":execution_wrappers",
        ":telemetry_tools",
        ":layer_buildfarm_worker",
    ],
)

[
    oci_image_index(
        name = "buildfarm-%s" % image,
        images = [
            ":buildfarm-%s_linux_%s" % (image, arch)
            for arch in ARCH
        ],
        tags = ["container"],
    )
    for image in [
        "server",
        "worker",
    ]
]

######
# Helpers to write to the local Docker Desktop's registry
# Usage: `bazel run //:tarball_server_amd64 && docker run --rm buildfarm-server:amd64`
######
[
    [
        oci_tarball(
            name = "tarball_%s_%s" % (image, arch),
            image = ":buildfarm-%s_linux_%s" % (image, arch),
            repo_tags = ["buildfarm-%s:%s" % (image, arch)],
            tags = ["container"],
        ),
        # Below targets push public docker images to bazelbuild dockerhub.
        oci_push(
            name = "public_push_buildfarm-%s" % image,
            image = ":buildfarm-%s" % image,
            repository = "index.docker.io/bazelbuild/buildfarm-%s" % image,
            # Specify the tag with `bazel run public_push_buildfarm-server public_push_buildfarm-worker -- --tag latest`
            tags = ["container"],
        ),
    ]
    for arch in ARCH
    for image in [
        "server",
        "worker",
    ]
]
