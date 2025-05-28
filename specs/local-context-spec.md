# Client-hosted Context Specification

Status: `REVIEW`

Versions:
 - (Current) 1.0.0: Initial specification

## Summary

Client-hosted context is context available to AI systems while working within a client's environment. For example, code editors, IDEs, and other development tools often provide context to AI systems while working on a developer's computer. When working within this environment, this client-hosted context specification is how compliant AI systems should search for and find the proper context to use within its processes.

AI systems MUST use default client context paths such as `.context/*` to search for context files locally and globally. `.context/config.json` or `.context/config.yaml` MUST be reserved for use as additional context configuration (e.g. overriding defaults, adding additional context files, etc.).

## Goals

- Give end users of AI systems a reliable location and expectations for context files on their client environments.
- Allow AI systems to have ways to adapt and migrate to this standardized specification.
- Allow users and context providers ways to customize where and how context should be considered within nested directories for more nuanced use cases.
- Provide a consistent means for users and context providers to configure other context communication protocols (e.g. MCP) relative to projects and global on client systems.

## Specification scope

- AI systems reading context while working within a specific directory.
- AI systems reading context while working within nested directories.
- AI systems reading context that is globally available in the client environment.
- AI systems reading context that is available from dependencies.
- AI systems configuration for additional context communication protocols (e.g. MCP).

## Not in specification scope

- Specifying all context types AI systems may support and their formats.
- What AI systems decide to do with these context files to answer a user's query.
- The limits AI systems impose on the amount of context it processes within its own system.
- How often an AI system should search for context files.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in BCP 14 [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119) [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174) when, and only when, they appear in all capitals, as shown here.

### Terms and definitions

- **Context**: Files, data, rich media, etc. that are used to inform the work and attention of an AI system.
- **AI system**: agents, models, and tools that process context.
- **Client**: the environment in which an AI system is running. Generally, this is an end user's computer but it can also be a hosted container or such hosted systems where the context and the running AI system are on the same machine (e.g. a continuous integration server).
- **Client-hosted Context**: context that is available to an AI system while it is running within a client environment.
- **Mounted directory**: for AI systems that select a directory for which all available work is possible (such as an IDE selecting a folder to open the editor with), this directory is the
"mounted directory". While the AI working directory can change as the AI system determines what work to do within nested directories, this mounted directory stays the same within the AI system's session or lifecycle.
- **AI working directory**: the directory in which the AI system is preparing to or is actively working within. If an AI system supports having a mounted directory (like an IDE opening a coding folder), this mounted directory is the default AI working directory. If the AI system's mounted directory is `~/myproject/`, then the AI working directory is by default `~/myproject/`. If the AI system determines that it needs to add/edit a file within `nested/paths/`, the AI system's AI working directory changes to be `~/myproject/nested/paths/` during that task. The mounted directory stays the same but, depending on what the AI system's agentic decisions, the AI working directory MAY change during the course of the AI system's work.
- **Client context path**: the relative path string that is used to find context files and configurations.
- **Context configuration**: a configuration file that provides additional rules for searching and filtering context to an AI system while it is running within AI working directory.
- **Ancestor context**: When AI systems have a specific mounted directory (typically a project directory) and the AI working directory is at a nested directory, all context and configuration between these two directories is the ancestor context. For example, if the mounted directory is `~/myproject` and the AI working directory is `~/myproject/nested/paths/feature/` then context and context configuration that might exist within `feature`, `paths`, `nested`, `myproject` are the entirety of possible ancestor context.
- **Glob pattern**: A string pattern that is used to match file paths. This specification uses the [glob](https://en.wikipedia.org/wiki/Glob_(programming)) syntax inclusive of the `**` pattern. Glob patterns must also be treated as case insensitive.


### Scope requirements

If an AI system supports client-hosted context, it MUST support one or more of the following context scopes:

- `global context`: context available to the AI system across the client environment.
- `static directory context`: context available to the AI system within the mounted directory (typically the root of a project) or the AI system's working directory if it does not work within a mounted directory.
- `dynamic subdirectory context`: context available to the AI system within subdirectories of the AI system's working directory (typically subdirectories of a project). The context files and configuration are dynamically selected and merged based on discovering context files and configuration at or between the mounted directory and the AI systems working directory.

### General path requirements

AI systems MUST normalize all path separators for the client operating system and expand the home directory symbol (~) using standard OS conventions.

AI systems SHOULD avoid following symlinks by default to avoid unintentional context inclusion.

AI systems MUST respect client OS file permissions, skipping and (optionally) warning about files that cannot be read due to permissions.

### Client context path

Within all context scopes, AI systems MUST search for context files within the default client context path that is relative to the AI working directory.

By default, the client context path is `.context` directory. e.g. `<aiWorkingDir>/.context/*` is where context files can be found.

The environment variable `CLIENT_CONTEXT_PATH` MAY be set by the user in order to change the path searched for by AI systems across all context scopes. This variable's value MUST be a relative path to AI working directories. This variable's value MAY include nested directories (e.g `other/path/for/ai`) as well as path traversal segments (`../shared-context/`).

Whether using the default path or incorporating the client environment variable override, the context path for AI Systems will be `<aiWorkingDir>/<clientContextPath>/*`

The client context path's directory MAY contain 0 context files or an unknown high number of files. AI Systems SHOULD limit the amount of context files allowed to be parsed and used. This specification does not set what the limit should be however AI Systems SHOULD inform users when AI System limits have been hit.

The client context path's directory MAY have subdirectories of unknown depth. AI Systems SHOULD limit the amount of subdirectory levels searched to a reasonable amount to avoid performance issues. This subdirectory limit MUST NOT be less than 3 to allow reasonable user categorization of context files.

### Context configuration location

The configuration file's intent is to manage what context is being considered while the AI system is working within the AI working directory.

By default, if configuration files are defined, they MUST be nested under the client context path with the filename `context-config.json` and MUST contain either no content or JSON compliant context configuration.

The full paths for AI systems for these files are `<aiWorkingDir>/<clientContextPath>/context-config.json`.

Given the default client context path is `.context`, then configurations in the default location MUST be defined at `<aiWorkingDir>/.context/context-config.json`

Like other context files, context configuration MAY be nested within subdirectories of the AI working directory.

### Context configuration

The type signature of the configuration file is as follows:

```typescript
interface Configuration {
  clientContext?: {
    includeFiles?: string[];
    excludeFiles?: string[];
    ignoreGlobalContext?: boolean;
    ignoreAncestorContext?: boolean;
  };
  mcpServers?: {
    [serverName: string]: {
      command: string;
      type?: "stdio" | "sse" | "http";
      args?: string[];
      env?: Record<string, string>;
      disabled?: boolean;
    } | {
      url: string;
      headers?: Record<string, string>;
      disabled?: boolean;
    };
  };
}
```

- `clientContext` - OPTIONAL - object containing configuration specific to client-hosted context.
- `clientContext.includeFiles` - OPTIONAL - a string of glob patterns for context files that MUST be included in addition to what may already be found. These paths MUST be relative to the AI working directory. e.g. `['additional-context/**', '/**/Desktop/shared-context/**']`
- `clientContext.excludeFiles` - OPTIONAL - a string of glob patterns for context files that MUST NOT be included in the final set of context files. These paths MUST be relative to the AI working directory. e.g. `['**/private-context/**']`
- `clientContext.ignoreGlobalContext` - OPTIONAL - a boolean that specifies if all global context should be ignored while doing work within the AI working directory. Default MUST be false.
- `clientContext.ignoreAncestorContext` - OPTIONAL - a boolean that specifies if all ancestor context should be ignored while doing work within the AI working directory. Default MUST be false.
- `mcpServers` - OPTIONAL - object containing configuration specific to MCP servers.

Where `clientContext.includeFiles` and `clientContext.excludeFiles` overlap, the `excludeFiles` pattern takes precedence and any matching files MUST be excluded.

When some or all fields are missing, the default configuration MUST apply the following values respectively:

```json
{
  "clientContext": {
    "includeFiles": [
      "*"
    ],
    "excludeFiles": [
      "context-config.json"
    ],
    "ignoreGlobalContext": false,
    "ignoreAncestorContext": false
  }
}
```

AI systems MAY add more to the default configuration but MUST NOT conflict with the expected defaults above.

AI systems MUST filter context file types that they do not support themselves and SHOULD show a warning message where there are selected context files that are not supported by the AI system.

AI systems MUST validate the configuration file based on the above type signature and SHOULD show a warning message where there are configuration files that have invalid configuration. AI systems MUST NOT fail to start or work because of invalid configuration.

### Merging algorithm for context and configuration

The merging algorithm for context and configuration is as follows:
1. source all context files and context configuration files based on the supported context scopes (global, static directory, dynamic subdirectory).
2. create a list of absolute paths for context files and track which files are from which context scope.
3. order all context configuration files based on the following precedence order where the closer to the AI working directory the configuration is, the higher priority it has:

    1. global context configuration (least priority)
    2. mounted directory context configuration
    3. each ancestor context configuration that exists from leftmost directory to rightmost directory (most priority)
4. recursively merge all context configuration files to generate the final configuration to apply
    1. `clientContext.includeFiles` - arrays should be appended to and deduplicated based on pattern
    2. `clientContext.excludeFiles` - arrays should be appended to and deduplicated based on pattern
    3. `clientContext.ignoreGlobalContext` - the last value wins
    4. `clientContext.ignoreAncestorContext` - the last value wins
    5. `mcpServers` - arrays should be appended to and deduplicated based on server name
5. apply the final configuration to the AI system
    1. `clientContext.includeFiles` - these should be used to source any new context files to add to the current list of context files (from step 2)
    2. `clientContext.excludeFiles` - these should be used to remove any context files from the final list of context files. File exclusion MUST come after file inclusion as it's a higher precedence operation.
    3. `clientContext.ignoreGlobalContext` - if the final result if `true` then the AI system should drop all global context files from the final list of context files. if `false` do nothing.
    4. `clientContext.ignoreAncestorContext` - if the final result if `true` then the AI system should drop all ancestor context files from the final list of context files. if `false` do nothing.
    5. `mcpServers` - start up any MCP servers not already started and make them available within the available context sources and tool calls.


### Explicit context exclusion

To avoid inadvertent exposure of sensitive information, AI systems MUST explicitly exclude files that are known to have very sensitive information. At very minimum, this list of patterns MUST be used to always exclude files: `.env`, `.env.*`, `.key`, `.key.*`, `.pem`, `.pem.*`, `.crt`, `.crt.*`, `.p12`, `.p12.*`, `.pfx`, `.pfx.*`, `.jks`, `.jks.*`, `.keystore`, `.keystore.*`, `.ppk`, `.ppk.*`, `.ssh/id_*`, `.kdbx`, `.kdbx.*`, `.asc`, `.asc.*`, `.gpg`, `.gpg.*`, `credentials*`, `*_key`, `*_key.*`, `.ovpn`, `.ovpn.*`.


### Global context

Global context is used for setting context files and context configuration in a way that should be discoverable and used in all AI system work unless where configured to be ignored.

By default, global context should be sourced from the client environment home directory. Therefore, default global context path is `<homeDir>/<clientContextPath>` (e.g. `~/.context/*`).

Users MAY specify a client environment variable `GLOBAL_CONTEXT_PATH` which is the absolute path to the location of globally set context files and context configuration. This path MUST NOT include the client context path.

Example, if I set the `GLOBAL_CONTEXT_PATH` to `/Users/myuser/Desktop/shared-context` then the AI system will look for the client context path within this directory (`/Users/myuser/Desktop/shared-context/<clientContextPath>`). If the client context path is still the default, then `/Users/myuser/Desktop/shared-context/.context/*` is where I must put my global context files.

Anticipating user error with `GLOBAL_CONTEXT_PATH`, AI systems SHOULD attempt to identify if the `GLOBAL_CONTEXT_PATH` includes the client context path and gracefully handling using that as the full directory.

Because the global context path can include configuration, including context communication protocols, AI systems MUST use this configuration for global context configuration. For example, in addition to AI system-specific MCP server definition files, all MCP servers configured at the global context path should be incorporated into the list of all configured MCP servers.

### Static directory context

Static directory context is sourcing context files and context configuration from a pre-determined location and not changing the context during the active work of the AI system.

For static directory context support, AI systems MUST source context files and configuration from the root of the mounted directory or AI working directory if mounted directory is not supported by the AI system.

Example, if the mounted directory is `~/myproject` then the AI system must look for context and configuration at `~/myproject/<clientContextPath>`.

### Dynamic subdirectory context

Dynamic subdirectory context is actively adjusting available context to the AI system based on the active work that the AI system is doing and where it is doing it. This method allows context to be colocated to where work is being done.

As AI systems determine work to do, they select which files/folders to perform actions (e.g. edit a file or create a file in a directory). When the AI system selects where work needs to happen, the rightmost directory on the path becomes the AI working directory. The directories between this AI working directory and the mounted directory are the ancestor directories. Each of those directories must be sourced for context files and configuration.

For example, if the mounted directory is `~/myproject` and the AI working directory is `~/myproject/src/components` then the AI system must look for context files and configuration at `~/myproject/src/components/.context`, `~/myproject/src/.context`, and `~/myproject/.context`.

AI systems MAY cache these lookups to improve performance for many directory lookups.


### Context file formats

There are no active standards around context file formats. The following is the industry best current practices for context files formats while improved standards are being formalized.

- **Plaintext files**
  - `.txt` files represent plaintext files. AI systems MUST accept the raw context as context.

- **Markdown files**
  - `.md` and `.mdc` files represent markdown files. AI systems MUST accept both extensions and the raw context as context. If the file contains [front matter](https://jekyllrb.com/docs/front-matter/), AI systems MUST parse the front matter and use it as properties.


#### Context file properties

AI Systems MUST use context file properties to filter down the applicable use of context.

Properties individual context files MAY set:
- `description` - OPTIONAL - a description of the context file and where it would be most applicable. Default is empty string. A description MUST be set if the trigger is `agent`.
- `globs` - OPTIONAL - an array of glob patterns that define the files that the context file is applicable to. Default is `[*]` (all files).
- `disabled` - OPTIONAL - a boolean value that indicates if the context file should be ignored. Default is `false`.
- `trigger` - OPTIONAL - a string that defines the trigger for the context file. Default is empty string. AI systems MUST compare trigger values with a case insensitive comparison. Default is `manual`.
  - `always` - Always included in the model context.
  - `auto` - Included when files matching a glob pattern are referenced.
  - `agent` - Available to the AI, which decides whether to include it. User MUST provide a description.
  - `manual` - Only included when explicitly mentioned using a AI system specific reference.

AI systems MAY accept context files with properties other than the ones defined above for backwards compatibility.

Example context files:

```md
---
description: "Format preferences for JavaScript files"
globs: **/*.{js|ts}, webpackconfig.js
trigger: auto
---

Keep the modules in alphabetical order but grouped by external import vs internal imports.

```

```md
---
description: "Format preferences for CSS files"
globs:
  - **/*.{css}
  - **/*.{scss|less}
trigger: auto
---

Don't use @import for CSS files.

```

AI Systems MAY truncate context files based on their length and the scenarios where they are used.


## Migration considerations

There are many existing path expectations for clients today. To support a graceful migration, this specification allows clients to continue using the existing paths and formats while also allowing for the new paths to be used. It will also allow end users to specify any overrides such that any existing client-hosted context paths should continue to work by providing a `.context/context-config.json` file that has an `clientContext.includeFiles` property that points to the existing context files.

AI systems MAY continue using existing paths and properties that map to this specification for backwards compatibility.

For example, if a system currently uses `.myairules` as a file or `.myai/rules/*` these can be incorporated into the AI system's default configuration as follows:

```json
{
  "include": [
    ".myairules",
    ".myai/rules/*"
  ]
}
```
