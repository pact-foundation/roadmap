---
name: configuration-and-shared-files
started: 2024-08-30
pr: pact-foundation/roadmap#0000
tracking_issue: pact-foundation/roadmap#0000
---
## Summary

This RFC proposes to establish a standard way to configure and share common files across the Pact ecosystem, including the Pact Broker, PactFlow, and the various Pact implementations.

## Motivation

The Pact ecosystem is quite diverse and we would like to ensure that transitioning between different tools is as seamless as possible. Several de-facto standards have emerged, but these have not been formalised. Furthermore, there is no clear way to extend these de-facto standards to new tools or use cases.

This RFC aims to formalise some existing de-facto standards. Furthermore, this RFC proposes new standards with the hope of ensuring a more consistent experience across the Pact ecosystem and into the future.

This RFC is split into three main categories:

- **Environment Variables**: Define a standard way to configure Pact implementations using environment variables.
- **Configuration**: Define a standard way to store runtime configuration for Pact implementations.
- **Shared Files**: Define a standard way to store shared files across the Pact ecosystem.

## Environment Variables

Environment variables are a common way to configure tools, especially when running in a containerised environment. The Pact ecosystem already uses `PACT_BROKER_*` environment variables to configure how the client connects to the Pact Broker. This RFC proposes to formalise this standard, as well as extend it to other areas of the Pact ecosystem.

The following environment variables are proposed:

- **`PACT_CONFIG_FILE`**

  Defines the location of the configuration file to use. This will override any other configuration files that are found. If the file does not exist, an error _must_ be raised.

  If this variable is not set, the implementation _must_ look for a local configuration file in the project directory, and then a user configuration file.

- **`PACT_DATA_HOME`**

  Defines the directory where shared files are stored. This is used by the Pact implementations to store shared files.

- **`PACT_PLUGIN_DIR`**

  Defines the directory where shared plugins are stored. This is used by the Pact implementations to load shared plugins. This should default to `$PACT_DATA_HOME/plugins` if not set.

- **`PACT_DO_NOT_TRACK`**

  If the environment variable is set (regardless of its value), the Pact implementation _must not_ report any usage analytics data back to the Pact Foundation.

### Pact Broker

When a Pact implementation needs to connect to a Pact Broker, it should use the following environment variables (which are already widely used across the Pact ecosystem):

- **`PACT_BROKER_BASE_URL`**

  Defines the base URL of the Pact Broker. This _must_ include the protocol (`http`/`https`), port if non-standard, and path if not at the root of the domain.

  The base URL _may_ contain a username and password, but end users should prefer to use the `PACT_BROKER_USERNAME` and `PACT_BROKER_PASSWORD` environment variables instead.

  Examples:

  - `https://pact-broker.local.company.com:9292`
  - `http://localhost:9292/pact-broker/`
  - `https://user:password@localhost:12345`

- **`PACT_BROKER_USERNAME`**, **`PACT_BROKER_PASSWORD`**

  Defines the username and password respectively to use when authenticating with the Pact Broker. The Pact client _must_ use these values to perform a [Basic authorisation](https://datatracker.ietf.org/doc/html/rfc7617) with the Pact Broker.

  Having either of these variables set alongside a `PACT_BROKER_TOKEN` is undefined behaviour.

- **`PACT_BROKER_TOKEN`**

  Defines a token to use when authenticating with the Pact Broker. The Pact client _must_ use this value to perform a Bearer authorisation with the Pact Broker as per [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750).

  Having this variable set alongside a `PACT_BROKER_USERNAME` and/or `PACT_BROKER_PASSWORD` is undefined behaviour.

The Pact Broker itself also uses a large number of environment variables to configure its behaviour; however, these are specific to the Pact Broker and fall under the [Implementation Specific](#implementation-specific) category.

### Implementation Specific

Implementation specific environment variables are left to the discretion of the implementation or tool. The RFC suggests that these variables should be prefixed with `PACT_{IDENT}_*`, where `{IDENT}` is an uppercase identifier for the language or tooling being used. The variable name should be in uppercase with underscores separating words.

Examples:

- `PACT_BROKER_BASIC_AUTH_READ_ONLY_USERNAME`
- `PACT_NET_DISABLE_FOO`
- `PACT_RUBY_BAR`

It is up to the discretion of the language implementation to define the meaning of these variables. The implementation or tool may also deviate from this standard if deemed necessary or in order to follow language-specific conventions.

The Pact Broker provides a good example of this guideline, where it uses `PACT_BROKER_*` to allow for many aspects of the broker to be configured. See the [Pact Broker Configuration docs](https://github.com/pact-foundation/pact_broker/blob/master/docs/CONFIGURATION.md) for more information of this use case.

### Configuration

This RFC proposes the use of [TOML](https://toml.io/en/) configuration files for storing configuration data. TOML is chosen due to its simplicity and readability. Other formats such as JSON or YAML could have been chosen, but they have the following drawbacks:

- JSON is less human-friendly than TOML and does not support comments which can be useful for documenting configuration files.
- YAML is more complex than TOML, and can be harder to parse correctly. It is also prone to errors due to the way it handles indentation and somewhat unconventional types (e.g., sexagesimal numbers where `1:2:3` equals `3723`, octal numbers where `0123` equals `83`, and automatic conversion of boolean-like values such as `no`, `on`, `y` into booleans).

Configuration files complement the use of environment variables. They can allow for more complex configuration data to be stored, and can be used to store default values shared across the Pact ecosystem without requiring the end user to set environment variables all the time.

This RFC also proposes to have both local (or per-project) configuration files, and user configuration files. This allows a user of Pact to configure their environment in a way that is consistent across all projects, as well as allowing for project-specific configuration.

For example:

- A user may wish to set the their default Pact Broker to `https://pact-broker.local.company.com` in their user configuration file so that all projects use this value by default.
- A specific project might still be in development, and use a local Pact Broker at `http://localhost:9292` for testing. This could be set in the project's local configuration file.
- The CI/CD pipeline still configures the Pact Broker using environment variables, as it is not practical to store configuration files in the pipeline.

#### Precedence

The following order of precedence _must_ be followed when reading configuration data:

1. CLI arguments (if applicable)
2. Environment variables
3. Configuration files, in the following order:
   1. If an explicit configuration file is set, only that file should be read.
   2. If no explicit configuration file is set, the following files should be read:

      1. Local configuration file
      2. User configuration file

      The options are merged, with the local configuration file taking precedence over the user configuration file.
4. Default values

The explicit configuration file may either be set by the `PACT_CONFIG_FILE` environment variable, or by a command line option.

#### Location

To ensure a consistent experience across the Pact ecosystem, the following configuration files are proposed to be used across the Pact ecosystem:

- **User Configuration File**

  Location:

  - On all systems, if the configuration file is explicitly set (either by `PACT_CONFIG_FILE` or a command line option), this location _must_ be used.
  - Unix and Unix-like systems:
    - `$XDG_CONFIG_HOME/pact/config.toml` (which defaults to `~/.config/pact/config.toml` if `XDG_CONFIG_HOME` is not set)
  - Windows:
    - `%APPDATA%\pact\config.toml`
  - Fallback:
    - `~/.pact/config.toml`

  > [!NOTE]
  >
  > macOS is considered a Unix-like system for the purposes of this RFC. Should an end-user prefer to use a different location (such as `~/Library/Preferences`), they may do so by setting the `XDG_CONFIG_HOME` environment variable.

  If the operating system cannot be detected, the fallback location _should_ be used.

  If the operating system is detected and the configuration file is found in the fallback location, a warning _should_ be displayed informing the user to relocate the configuration. The exception to this is if the location of the config file is explicitly set, in which case the warning _must not_ be displayed.

  If an end user has multiple user configuration files, it is undefined behaviour.

- **Local Configuration File**

  Location:

  - In the process's working directory, either of:
    - `pact.toml`
    - `.pact.toml`
  - Or looking up the directory tree for the first of the above files.

  If both `pact.toml` and `.pact.toml` are found in the same directory, it is undefined behaviour.

  The parsing of the configuration file _must_ stop at the first file found.

#### Format

The TOML configuration file _must_ be organised into sections (or tables) to group configuration data. Top-level configuration data _must not_ be allowed.

Keys should be in lowercase, and use hyphens (`-`) to separate words. This is to ensure a consistent experience across the Pact ecosystem.

The following tables are proposed:

- `pact`

  To store configuration that is shared across the Pact ecosystem:

  ```toml
  [pact]
  plugin-dir = "~/.local/share/pact/plugins"
  data-home = "~/.local/share/pact"
  do-not-track = true
  ```

  The following entries are proposed as part of this RFC:

  - `plugin-dir`: The directory where shared plugins are stored. Equivalent to `PACT_PLUGIN_DIR`.
  - `data-home`: The directory where shared files are stored. Equivalent to `PACT_DATA_HOME`.
  - `do-not-track`: If set to `true`, the Pact implementation _must not_ report any usage analytics data back to the Pact Foundation.

- `broker`

  To store configuration as to how to connect to the Pact Broker:

  ```toml
  [broker]
  base-url = "https://pact-broker.local.company.com:9292"
  auth = { username = "user", password = "password" }
  # or
  auth = { token = "token" }
  ```

  The following entries are proposed:

  - `base-url`: The base URL of the Pact Broker. Equivalent to `PACT_BROKER_BASE_URL`.
  - `auth`: The authentication details to use when connecting to the Pact Broker. This is a sub-table with the following entries:

    - `auth.username`: The username to use when authenticating with the Pact Broker. Equivalent to `PACT_BROKER_USERNAME`.
    - `auth.password`: The password to use when authenticating with the Pact Broker. Equivalent to `PACT_BROKER_PASSWORD`.
    - `auth.token`: The token to use when authenticating with the Pact Broker. Equivalent to `PACT_BROKER_TOKEN`.

    <!-- comment about how override should treat auth as a block -->
    Note that the `auth` block should be treated as a single entity. If any of the `auth` fields are set in a high precedence configuration, they should override the entire `auth` block of a lower precedence configuration.

  The Pact Broker itself _may_ have additional configuration options within the `broker` table.

- `{ident}`

  To store configuration specific to a particular language implementation or tool. It is up to the implementation or tool to define the meaning of these entries, and each implementation or tool should only read entries under its own identifier.

  Sub-tables may used to group configuration data within each `{ident}` section.

  ```toml
  [ruby]
  bar = "baz"

  [net.foo]
  disabled = true

  [pactflow]
  enable-other-feature = false
  ```

#### Path Handling

To ensure a uniform experience across the Pact ecosystem and operating systems, the following rules _must_ be followed when handling paths in the configuration file:

- Paths _must_ be stored using the forward slash (`/`) as the path separator.
  - This avoids the need to escape backslashes (`\`) in Windows paths.
  - If a backslash is used, it is undefined behaviour. The implementation _may_ attempt to convert the backslashes to forward slashes, or may use the value as-is.
- The tilde (`~`) character _must_ be expanded to the user's home directory if it is the first character in a path.
  - For example, parsing `~/.local/share/pact` should result in `/home/user/.local/share/pact` on Unix-like systems.
  - If the tilde appears elsewhere in the path, it is treated as a literal character.
  - If someone wishes to use a literal tilde for a relative path, they _must_ use `./~` instead.
- Relative paths are allowed, and _must_ be resolved relative to the configuration file.
  - For example, if the configuration file is located at `/workdir/project/.pact.toml`, and a value is set to `.pact/data`, the resolved path should be `/workdir/project/.pact/data`.


### Shared Files

At present, there are only a few files which are shared across the Pact ecosystem. These are:

- Plugins, currently stored in `~/.pact/plugins`
- The Pact core library (i.e., the Pact FFI), though only in some implementations.

As with the configuration files, a standard location is proposed for shared files which follows the operating system conventions:

- If the `PACT_DATA_HOME` environment variable is set, this location _must_ be used.
- Unix and Unix-like systems: `$XDG_DATA_HOME/pact/` (which defaults to `~/.local/share/pact/` if `XDG_DATA_HOME` is not set)
- Windows: `%APPDATA%\pact\`
- Fallback: `~/.pact/`

> [!NOTE]
>
> macOS is considered a Unix-like system for the purposes of this RFC. Should an end-user prefer to use a different location (such as `~/Library/pact/`), they may do so by setting the `XDG_DATA_HOME` environment variable.

If the operating system cannot be detected, the fallback location _should_ be used.

If the operating system is detected and the fallback location is found, a warning _should_ be displayed to inform the user to relocate the directory. This exception to this is if the location of the shared files is explicitly set, in which case the warning _must not_ be displayed.

## Drawbacks

- This RFC may be seen as too prescriptive, and may not be flexible enough for some use cases. Hopefully enough flexibility has been built in to cater for most use cases, but it is possible that some edge cases may not be covered.
- There is an existing de-facto standard to use `~/.pact/`. This RFC proposes a backwards compatible way to ensure that users can migrate to the new standard without breaking existing setups, but this will unfortunately require some additional work on behalf of the maintainers of the various Pact implementations to ensure that this is handled correctly.

  If an end user wishes to keep using `~/.pact`, they should:

  1. Set `PACT_CONFIG_FILE` to `~/.pact/config.toml`
  2. Either:
     1. Set `PACT_DATA_HOME` to `~/.pact/`
     2. Add the following to `~/.pact/config.toml`:

        ```toml
        [pact]
        data-home = "."
        ```

## Rationale and alternatives

- The main rationale for implementing this RFC is to ensure a consistent experience across the Pact ecosystem. The creation of configuration files provides a simpler end-user experience which is more flexible than environment variables alone.
- It is possible to maintain the status quo and not implement this RFC. While this may not impact the use of environment variables such as `PACT_BROKER_BASE_URL` which are already widely used, it may make it harder to transition to a more configuration-file based approach in the future. Furthermore, as other parts of the Pact ecosystem are developed, it may be harder to ensure a consistent experience across the ecosystem.

## Unresolved questions

> [!NOTE]
>
> At present, there are no unresolved questions, though I expect that there may be some as this RFC is discussed.

## Future possibilities

The RFC has been proposed in a way that should allow for future expansion. In particular:

- New environment variables can be added as needed. The `PACT_{IDENT}_*` prefix allows for new variables to be added without clashing with existing variables.
- The TOML configuration file format is flexible enough to allow for new sections to be added as needed without impacting existing implementations.
- By specifying a standard location for shared files, implementations can leverage this in the future to share additional files across the Pact ecosystem.
