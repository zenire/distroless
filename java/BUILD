load("@contrib_rules_oci//oci:defs.bzl", "oci_image", "oci_image_index", "structure_test")
load("@rules_pkg//:pkg.bzl", "pkg_tar")
load("//private/oci:defs.bzl", "java_image")
load("//base:base.bzl", "deb_pkg")
load("//base:distro.bzl", "DISTROS")
load("//cacerts:java.bzl", "cacerts_java")
load("//java:jre_ver.bzl", "jre_ver")
load("//:debian_versions.bzl", DEBIAN_VERSIONS = "DEBIAN_PACKAGE_VERSIONS")
load("//:java_archives.bzl", "JAVA_RELEASE_VERSIONS")

package(default_visibility = ["//visibility:public"])

USERS = [
    "root",
    "nonroot",
]

JAVA_ARCHITECTURES = [
    "amd64",
    "arm64",
    "s390x",
    "ppc64le",
]

JAVA_21_ARCHITECTURES = [
    "amd64",
    "arm64",
    # "s390x", adoptium doesn't have a build yet
    "ppc64le",
]

JAVA_VERSIONS_PER_DISTRO = [
    ("11", "debian11"),
    ("17", "debian11"),
    ("17", "debian12"),
]

[
    cacerts_java(
        name = "cacerts_java_" + arch + "_" + distro,
        cacerts_tar = "//base:cacerts_" + arch + "_" + distro,
    )
    for distro in DISTROS
    for arch in JAVA_ARCHITECTURES
]

[
    pkg_tar(
        name = "openjdk_jre_headless_" + java_version + "_" + arch + "_" + distro,
        symlinks = {
            "/usr/bin/java": "/usr/lib/jvm/java-" + java_version + "-openjdk-" + arch + "/bin/java",
        },
        deps = [
            deb_pkg(
                arch,
                distro,
                "openjdk-" + java_version + "-jre-headless",
            ),
        ],
    )
    for arch in JAVA_ARCHITECTURES
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO
]

# special case for java 21, we will start using temurin, the goal is to slowly transition all builds
# to temurin, to also back support java 11 on debian12.
[
    pkg_tar(
        name = "temurin_jre_" + java_version + "_" + arch,
        symlinks = {
            "usr/bin/java": "/usr/lib/jvm/temurin" + java_version + "_jre_" + arch + "/bin/java",
        },
        deps = [
            "@temurin" + java_version + "_jre_" + arch,
        ],
    )
    for arch in JAVA_21_ARCHITECTURES
    for java_version in ["21"]
]

[
    pkg_tar(
        name = "temurin_jdk_" + java_version + "_" + arch,
        symlinks = {
            "usr/bin/java": "/usr/lib/jvm/temurin" + java_version + "_jdk_" + arch + "/bin/java",
        },
        deps = [
            "@temurin" + java_version + "_jdk_" + arch,
        ],
    )
    for arch in JAVA_21_ARCHITECTURES
    for java_version in ["21"]
]

# Base
[
    oci_image_index(
        name = "java_base" + mode + "_" + user + "_" + distro,
        images = [
            "java_base" + mode + "_" + user + "_" + arch + "_" + distro
            for arch in JAVA_ARCHITECTURES
        ],
    )
    for mode in [
        "",
        "_debug",
    ]
    for user in USERS
    for distro in DISTROS
]

# a build-base that isn't published, but used to generate our publishable base and as a base for temurin
# temurin ships with libharfbuzz and cacerts but debian distributed ones do not include it
# libharfbuzz depends on libpcre,libglib,libgraphite
[
    oci_image(
        name = "java_build_base" + mode + "_" + user + "_" + arch + "_" + distro,
        base = ("//base:base_nossl" if (not ("debug" in mode)) else "//base:base_nossl_debug") + "_" + user + "_" + arch + "_" + distro,
        env = {"LANG": "C.UTF-8"},
        tars = [
            deb_pkg(arch, distro, "zlib1g"),
            deb_pkg(arch, distro, "libjpeg62-turbo"),
            deb_pkg(arch, distro, "libpng16-16"),
            deb_pkg(arch, distro, "liblcms2-2"),
            deb_pkg(arch, distro, "libfreetype6"),
            deb_pkg(arch, distro, "fonts-dejavu-core"),
            deb_pkg(arch, distro, "fontconfig-config"),
            deb_pkg(arch, distro, "libexpat1"),
            deb_pkg(arch, distro, "libfontconfig1"),
            deb_pkg(arch, distro, "libuuid1"),
            deb_pkg(arch, distro, "libbrotli1"),
            deb_pkg(arch, distro, "libcrypt1"),
            deb_pkg(arch, distro, "libstdcpp6"),
            deb_pkg(arch, distro, "libgcc-s1"),
            "//locale:locale_" + arch + "_" + distro,
        ],
    )
    for mode in [
        "",
        "_debug",
    ]
    for user in USERS
    for distro in DISTROS
    for arch in JAVA_ARCHITECTURES
]

# libpcre is a dependency for libharfbuzz (so this is specifically for common java_base)
DISTRO_SPECIFIC_LIBRARIES = {
    "debian11": [
        "libpcre3",
    ],
    "debian12": [
        "libpcre2-8-0",
    ],
}

[
    oci_image(
        name = "java_base" + mode + "_" + user + "_" + arch + "_" + distro,
        base = "java_build_base" + mode + "_" + user + "_" + arch + "_" + distro,
        env = {"LANG": "C.UTF-8"},
        tars = [
            deb_pkg(arch, distro, "libharfbuzz0b"),
            deb_pkg(arch, distro, "libgraphite2-3"),
            deb_pkg(arch, distro, "libglib2.0-0"),
            ":cacerts_java_" + arch + "_" + distro,
        ] + [deb_pkg(arch, distro, library) for library in DISTRO_SPECIFIC_LIBRARIES[distro]],
    )
    for mode in [
        "",
        "_debug",
    ]
    for user in USERS
    for distro in DISTROS
    for arch in JAVA_ARCHITECTURES
]

# Java JRE Headless
[
    oci_image_index(
        name = "java" + java_version + "_" + user + "_" + distro,
        images = [
            "java" + java_version + "_" + user + "_" + arch + "_" + distro
            for arch in JAVA_ARCHITECTURES
        ],
    )
    for user in USERS
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO
]

[
    oci_image(
        name = "java" + java_version + "_" + user + "_" + arch + "_" + distro,
        base = ":java_base_" + user + "_" + arch + "_" + distro,
        # We expect users to use:
        # cmd = ["/path/to/deploy.jar", "--option1", ...]
        entrypoint = [
            "/usr/bin/java",
            "-jar",
        ],
        env = {
            "JAVA_VERSION": jre_ver(DEBIAN_VERSIONS[arch][distro]["openjdk-" + java_version + "-jre-headless"]),
        },
        tars = [
            ":openjdk_jre_headless_" + java_version + "_" + arch + "_" + distro,
        ],
    )
    for arch in JAVA_ARCHITECTURES
    for user in USERS
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO
]

# Java JRE Headless - Debug
[
    oci_image_index(
        name = "java" + java_version + "_debug_" + user + "_" + distro,
        images = [
            "java" + java_version + "_debug_" + user + "_" + arch + "_" + distro
            for arch in JAVA_ARCHITECTURES
        ],
    )
    for user in USERS
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO
]

[
    oci_image(
        name = "java" + java_version + "_debug_" + user + "_" + arch + "_" + distro,
        architecture = arch,
        base = ":java_base_debug_" + user + "_" + arch + "_" + distro,
        # We expect users to use:
        # cmd = ["/path/to/deploy.jar", "--option1", ...]
        entrypoint = [
            "/usr/bin/java",
            "-jar",
        ],
        env = {
            "JAVA_VERSION": jre_ver(DEBIAN_VERSIONS[arch][distro]["openjdk-" + java_version + "-jre-headless"]),
        },
        tars = [
            ":openjdk_jre_headless_" + java_version + "_" + arch + "_" + distro,
            deb_pkg(
                arch,
                distro,
                "openjdk-" + java_version + "-jdk-headless",
            ),
        ],
    )
    for user in USERS
    for arch in JAVA_ARCHITECTURES
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO
]

# Temurin Java 21
[
    oci_image_index(
        name = "java" + java_version + "_" + user + "_" + distro,
        images = [
            "java" + java_version + "_" + user + "_" + arch + "_" + distro
            for arch in JAVA_21_ARCHITECTURES
        ],
    )
    for user in USERS
    for java_version, distro in [("21", "debian12")]
]

[
    oci_image(
        name = "java" + java_version + "_" + user + "_" + arch + "_" + distro,
        base = ":java_build_base_" + user + "_" + arch + "_" + distro,
        # We expect users to use:
        # cmd = ["/path/to/deploy.jar", "--option1", ...]
        entrypoint = [
            "/usr/bin/java",
            "-jar",
        ],
        env = {
            "JAVA_VERSION": JAVA_RELEASE_VERSIONS["temurin" + java_version + "_jre_" + arch],
        },
        tars = [
            ":temurin_jre_" + java_version + "_" + arch,
        ],
    )
    for arch in JAVA_21_ARCHITECTURES
    for user in USERS
    for java_version, distro in [("21", "debian12")]
]

# Temurin Java 21 Debug
[
    oci_image_index(
        name = "java" + java_version + "_debug_" + user + "_" + distro,
        images = [
            "java" + java_version + "_debug_" + user + "_" + arch + "_" + distro
            for arch in JAVA_21_ARCHITECTURES
        ],
    )
    for user in USERS
    for java_version, distro in [("21", "debian12")]
]

[
    oci_image(
        name = "java" + java_version + "_debug_" + user + "_" + arch + "_" + distro,
        architecture = arch,
        base = ":java_build_base_debug_" + user + "_" + arch + "_" + distro,
        # We expect users to use:
        # cmd = ["/path/to/deploy.jar", "--option1", ...]
        entrypoint = [
            "/usr/bin/java",
            "-jar",
        ],
        env = {
            "JAVA_VERSION": JAVA_RELEASE_VERSIONS["temurin" + java_version + "_jdk_" + arch],
        },
        tars = [
            ":temurin_jdk_" + java_version + "_" + arch,
        ],
    )
    for user in USERS
    for arch in JAVA_21_ARCHITECTURES
    for java_version, distro in [("21", "debian12")]
]

[
    structure_test(
        name = "java_base" + mode + "_" + user + "_" + arch + "_" + distro + "_test",
        config = ["testdata/java_base" + mode + ".yaml"],
        image = ":java_base" + mode + "_" + user + "_" + arch + "_" + distro,
        tags = [
            arch,
            "manual",
        ],
    )
    for mode in [
        "",
        "_debug",
    ]
    for arch in JAVA_ARCHITECTURES
    for user in USERS
    for distro in DISTROS
]

[
    structure_test(
        name = "java" + java_version + "_" + user + "_" + arch + "_" + distro + "_test",
        config = ["testdata/java" + java_version + "_" + distro + ".yaml"],
        image = ":java" + java_version + "_" + user + "_" + arch + "_" + distro,
        tags = [
            arch,
            "manual",
        ],
    )
    for user in USERS
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO + [("21", "debian12")]
    for arch in (JAVA_ARCHITECTURES if java_version != "21" else JAVA_21_ARCHITECTURES)
]

[
    structure_test(
        name = "java" + java_version + "_debug_" + user + "_" + arch + "_" + distro + "_test",
        config = ["testdata/java" + java_version + "_debug" + "_" + distro + ".yaml"],
        image = ":java" + java_version + "_debug_" + user + "_" + arch + "_" + distro,
        tags = [
            arch,
            "manual",
        ],
    )
    for user in USERS
    for java_version, distro in JAVA_VERSIONS_PER_DISTRO + [("21", "debian12")]
    for arch in (JAVA_ARCHITECTURES if java_version != "21" else JAVA_21_ARCHITECTURES)
]

RULE_NAMES = [
    ("java11_root_debian11", "java11_root_amd64_debian11"),
    ("java11_nonroot_debian11", "java11_nonroot_amd64_debian11"),
    ("java17_root_debian11", "java17_root_amd64_debian11"),
    ("java17_root_debian12", "java17_root_amd64_debian12"),
    ("java17_nonroot_debian11", "java17_nonroot_amd64_debian11"),
    ("java17_nonroot_debian12", "java17_nonroot_amd64_debian12"),
    ("java21_root_debian12", "java21_root_amd64_debian12"),
    ("java21_nonroot_debian12", "java21_nonroot_amd64_debian12"),
]

[
    java_image(
        name = "check_certs_" + rule_name,
        srcs = ["testdata/CheckCerts.java"],
        base = "//java:" + base_name,
        main_class = "testdata.CheckCerts",
    )
    for (rule_name, base_name) in RULE_NAMES
]

[
    structure_test(
        name = "check_certs_" + rule_name + "_test",
        config = ["testdata/java_certs.yaml"],
        image = ":check_certs_" + rule_name,
        tags = [
            "amd64",
            "manual",
        ],
    )
    for (rule_name, _) in RULE_NAMES
]

[
    java_image(
        name = "check_encoding_" + rule_name,
        srcs = ["testdata/CheckEncoding.java"],
        base = "//java:" + base_name,
        main_class = "testdata.CheckEncoding",
    )
    for (rule_name, base_name) in RULE_NAMES
]

[
    structure_test(
        name = "check_encoding_" + rule_name + "_test",
        config = ["testdata/java_encoding.yaml"],
        image = ":check_encoding_" + rule_name,
        tags = [
            "amd64",
            "manual",
        ],
    )
    for (rule_name, _) in RULE_NAMES
]

[
    java_image(
        name = "check_libharfbuzz_" + rule_name,
        srcs = ["testdata/CheckLibharfbuzz.java"],
        base = "//java:" + base_name,
        main_class = "testdata.CheckLibharfbuzz",
    )
    for (rule_name, base_name) in RULE_NAMES
]

[
    structure_test(
        name = "check_libharfbuzz_" + rule_name + "_test",
        config = ["testdata/java_libharfbuzz.yaml"],
        image = ":check_libharfbuzz_" + rule_name,
        tags = [
            "amd64",
            "manual",
        ],
    )
    for (rule_name, _) in RULE_NAMES
]
