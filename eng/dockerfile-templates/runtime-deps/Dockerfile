{{
    set dotnetVersion to join(slice(split(PRODUCT_VERSION, "."), 0, 2), ".") ^
    set isAlpine to find(OS_ARCH_HYPHENATED, "Alpine") >= 0 ^
    set isDebian to find(OS_ARCH_HYPHENATED, "Debian") >= 0 ^
    set isUbuntu to find(OS_ARCH_HYPHENATED, "Ubuntu") >= 0 ^
    set isFullMariner to defined(match(OS_VERSION, "^cbl-mariner\d+\.\d+$")) ^
    set isDistrolessMariner to defined(match(OS_VERSION, "^cbl-mariner\d+\.\d+-distroless$")) ^
    set isMariner to isFullMariner || isDistrolessMariner ^
    set distrolessStagingDir to "/staging" ^
    set marinerRegistry to when(isMariner && find(OS_VERSION, "2.0") >= 0,
        "cblmariner2preview.azurecr.io",
        "cblmariner.azurecr.io") ^
    set baseImageRepo to when(isAlpine,
        cat(ARCH_VERSIONED, "/alpine"),
        when(isDebian,
            cat(ARCH_VERSIONED, "/debian"),
            when(isUbuntu,
                cat(ARCH_VERSIONED, "/ubuntu"),
                when(isMariner,
                    cat(marinerRegistry, "/base/core"),
                    "<NOT-IMPLEMENTED>")))) ^
    set baseImageTag to when(isAlpine || isMariner, OS_VERSION_NUMBER, OS_VERSION)
}}{{
    if isDistrolessMariner:# Installer image
}}FROM {{baseImageRepo}}:{{baseImageTag}}{{if isDistrolessMariner: AS installer}}
{{ if isDistrolessMariner:
RUN {{InsertTemplate("../Dockerfile.linux.install-pkgs",
    [
        "pkgs": ["dnf"]
    ])}}
}}
{{if isDistrolessMariner:# Install .NET's dependencies into a staging location
}}RUN {{if isDistrolessMariner:mkdir {{distrolessStagingDir}} \
    && }}{{InsertTemplate("../Dockerfile.linux.install-deps", ["distroless-staging-dir": distrolessStagingDir])}}
{{ if isDistrolessMariner:
# Create a non-root user and group
RUN groupadd \
        --system \
        --gid=1000 \
        app \
    && adduser \
        --uid 1000 \
        --gid app \
        --shell /bin/false \
        --no-create-home \
        --system \
        app \
    # Copy user/group info to staging
    && cp /etc/passwd {{distrolessStagingDir}}/etc/passwd \
    && cp /etc/group {{distrolessStagingDir}}/etc/group

# Clean up staging
RUN rm -rf {{distrolessStagingDir}}/etc/dnf \
    && rm -rf {{distrolessStagingDir}}/run/* \
    && rm -rf {{distrolessStagingDir}}/var/cache/dnf \
    && find {{distrolessStagingDir}}/var/log -type f -size +0 -delete


# .NET runtime-deps image
FROM {{marinerRegistry}}/distroless/minimal:{{OS_VERSION_NUMBER}}

COPY --from=installer {{distrolessStagingDir}}/ /
^elif isFullMariner && ARCH_SHORT != "arm64":
# Install dotnet-runtime-deps package
RUN dotnet_version={{VARIABLES[cat("runtime|", dotnetVersion, "|build-version")]}} \
    && curl -fSL --output dotnet-runtime-deps.rpm {{VARIABLES[cat("base-url|", dotnetVersion, "|", VARIABLES["branch"])]}}/Runtime/$dotnet_version/dotnet-runtime-deps-$dotnet_version-cm.1-{{ARCH_SHORT}}.rpm \
    && dotnet_sha512='{{VARIABLES[cat("runtime-deps-cm.1|", dotnetVersion, "|linux-rpm|", ARCH_SHORT, "|sha")]}}' \
    && echo "$dotnet_sha512  dotnet-runtime-deps.rpm" | sha512sum -c - \
    && rpm --install dotnet-runtime-deps.rpm \
    && rm dotnet-runtime-deps.rpm
}}
{{InsertTemplate("../Dockerfile.common-dotnet-envs") ^
if isDistrolessMariner:

USER app}}
