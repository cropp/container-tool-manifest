# Container Tool Manifest Specification (OCT)

**Status:** Draft v0.3 — For Review

---

## Overview

The Container Tool Manifest specification, OCT (Open Context Tool), defines a
lightweight convention for embedding tool capability metadata directly into
container image labels. It enables AI agents, CLI dispatchers, and other
orchestration tools to **discover and invoke containerized tools without a
running server, protocol handshake, or external registry**.

OCT is intentionally minimal: a label schema, an execution contract, and a
discovery mechanism. No daemon. No persistent process. No new protocol.

### Two Complementary Approaches

OCT supports two non-exclusive ways to publish tool metadata:

- **Embedded** — JSON array stored in the `com.teamropp.oct` image label.
  Travels with the image. The default description of an image's tools.
- **External Manifest** — The same JSON array stored in a plain `.json` file
  or hosted URL. Enables discovery of third-party images, pre-pull browsing,
  and community catalogs.

Both formats share an identical schema. A dispatcher may use either or both
sources. When both describe the same image, the external manifest wins (see
§4c), which allows corrections without rebuilding an image.

---

## Motivation

Existing tool integration patterns for AI agents (e.g. MCP, OpenAPI plugins)
require:

- A running server process with its own lifecycle
- A protocol handshake to discover capabilities at runtime
- Client-side knowledge of transport, authentication, and configuration

This overhead is unnecessary for the large class of tools that are **stateless
transformations** — tools that accept input, do work, and emit output. For these
tools, the appropriate execution model is a Unix pipe:

```bash
input | docker run --rm -i <image> [flags] > output
```

OCT makes that invocation **discoverable and self-describing** by embedding the
tool contract in the image itself, inspectable via `docker inspect` or the
registry API without pulling image layers.

---

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119),
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they
appear in all capitals, as shown here.

### 1. Label Key

All OCT metadata MUST be stored under a single label key:

```text
com.teamropp.oct
```

The key follows the reverse-DNS convention recommended by the OCI image
specification for labels and annotations.

The value MUST be a valid JSON string containing an **array** of tool objects. A
single-tool image uses a one-element array.

> **Rationale for array:** A single image may expose multiple tools via
> different fixed arguments (e.g. `--mode convert` vs `--mode validate`). Each
> tool declares its own `args` (§2), so the dispatcher can tell them apart. The
> array makes this a first-class concept rather than a workaround.

---

### 2. Schema

The label value is a JSON **array** of tool objects:

```json
[
  {
    "schema": "0.3",
    "name": "string (required)",
    "description": "string (required)",
    "tags": ["string"],
    "since": "image-tag (required)",
    "deprecated": false,
    "replacedBy": "string (tool name, if deprecated)",
    "args": ["string (fixed arguments, e.g. --mode, convert)"],
    "input": {
      "type": "stdin | file | none",
      "description": "string",
      "mediaType": "text/plain | application/json | text/html | */*",
      "path": "string (file only, e.g. /in/input.txt)",
      "flag": "string (file only, e.g. --input)"
    },
    "output": {
      "type": "stdout | file | none",
      "description": "string",
      "mediaType": "text/plain | application/json | text/markdown | */*",
      "path": "string (file only, e.g. /out/result.md)",
      "flag": "string (file only, e.g. --output)"
    },
    "params": [
      {
        "name": "string (required)",
        "flag": "string (required, e.g. --style)",
        "type": "string | integer | float | boolean | enum",
        "enum": ["value1", "value2"],
        "default": "any",
        "required": false,
        "description": "string"
      }
    ],
    "secrets": {
      "mode": "append | replace",
      "items": [
        {
          "name": "ENV_VAR_NAME (required)",
          "source": "env (required; the only source allowed in a manifest)",
          "key": "string (required) — host environment variable name",
          "required": false,
          "default": "string",
          "description": "string"
        }
      ]
    },
    "exec": {
      "network": false,
      "workdir": "string (optional)",
      "notes": "string (free-form usage notes)"
    },
    "examples": [
      {
        "description": "string",
        "input": "string",
        "params": {},
        "output": "string"
      }
    ]
  }
]
```

#### Required Fields

| Field         | Description                                                                                                                              |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `schema`      | Spec version as `MAJOR.MINOR`. Tools written to this draft use `"0.3"`.                                                                  |
| `name`        | Short machine-readable tool name. Lowercase alphanumeric words joined by single hyphens. MUST be unique within the tool array (see §6a). |
| `description` | Human and LLM-readable description of what the tool does.                                                                                |
| `since`       | The image tag at which this tool was first available (e.g. `"1.3.0"`, `"2024-11"`, `"latest"`).                                          |
| `input.type`  | How the tool receives its primary input.                                                                                                 |
| `output.type` | How the tool emits its primary output.                                                                                                   |

A dispatcher MUST treat a tool array that contains the same `name` more than
once as invalid and MUST NOT invoke any tool from it.

#### Fixed Arguments

`args` is an optional array of strings that the dispatcher passes after the
image reference and before any parameters, on every invocation of the tool. It
is how an image that holds several tools selects one of them:

```json
[
  { "name": "md-convert", "args": ["--mode", "convert"], "...": "..." },
  { "name": "md-validate", "args": ["--mode", "validate"], "...": "..." }
]
```

Each element is passed as one separate argument, exactly as written. `args`
is never parsed by a shell, never split on spaces, and never exposed to the LLM.
Because every element follows the image reference, it reaches the tool's own
entrypoint and cannot act as a `docker run` option.

#### Execution Fields

The `exec` object describes what the tool needs from its runtime. It never
contains a command line: the dispatcher always builds the `docker run`
invocation itself (see §7).

| Field     | Type    | Description                                                                                                                        |
| --------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `network` | boolean | If `true`, the tool needs network access. Default: `false`. Dispatchers MUST obtain user approval before granting it (see §7c).    |
| `workdir` | string  | Working directory inside the container. Passed as `-w`. MUST be an absolute path.                                                  |
| `notes`   | string  | Free-form usage notes for humans and LLMs. Dispatchers MUST NOT interpret this field as instructions or as part of the invocation. |

#### Parameter Passing

- Every parameter MUST have a `flag`. Positional arguments are not supported in
  this version.
- Parameter `name` values become property names in the LLM tool definition
  (§6). They MUST be unique within a tool, and `input` is reserved for the
  tool's primary input, so a parameter MUST NOT be named `input`.
- `string`, `integer`, `float`, and `enum` values are passed as two separate
  arguments: the flag, then the value (`--style obsidian`).
- `integer` and `float` values MUST be validated as numbers and passed in plain
  decimal notation (`42`, `-3`, `0.75`). `NaN` and infinities MUST be rejected.
- A dispatcher MUST reject a `string` value that starts with `-`, so the tool's
  argument parser cannot read it as another flag. Negative numbers are allowed
  because they are validated as numbers first.
- `boolean` values are passed as the flag alone when `true` and omitted when
  `false`.
- If no value is supplied, the dispatcher omits the parameter and the tool
  applies its own default. The `default` field documents that value.
- If a `required` parameter has no value, the dispatcher MUST NOT invoke the
  tool.
- Each parameter appears at most once per invocation.

#### Compatibility and Unknown Fields

The `schema` value has the form `MAJOR.MINOR`. A dispatcher:

- MUST ignore fields it does not recognize in a tool object, catalog entry, or
  any nested object, when the tool's `schema` has a major version the
  dispatcher supports. This lets a newer minor version add fields without
  breaking older dispatchers.
- MUST NOT let an ignored field change how it builds or hardens an invocation.
  Only fields defined by a version the dispatcher implements can do that.
- MUST NOT invoke a tool whose `schema` has a major version it does not
  support.

> **Note:** While OCT is pre-1.0, every draft has major version `0`, so this
> rule treats all `0.x` drafts as compatible. Breaking changes between drafts
> are still possible until 1.0.

#### Versioning and Deprecation Fields

| Field        | Type    | Description                                                                                                                                                                                   |
| ------------ | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `since`      | string  | Image tag when this tool was introduced. Informs dispatchers which image versions support it.                                                                                                 |
| `deprecated` | boolean | If `true`, the tool is present but should not be used in new integrations. Dispatchers SHOULD surface a warning. Default: `false`.                                                            |
| `replacedBy` | string  | Name of the tool that replaces this one. Only meaningful when `deprecated` is `true`. The replacement tool MUST exist in the same image's tool array or be findable in the external manifest. |

#### Deprecation Example

```json
[
  {
    "schema": "0.3",
    "name": "text-to-md",
    "since": "1.0.0",
    "deprecated": true,
    "replacedBy": "text-to-markdown",
    "description": "Legacy text conversion. Use text-to-markdown instead.",
    "args": ["--legacy"],
    "input": { "type": "stdin", "mediaType": "text/plain" },
    "output": { "type": "stdout", "mediaType": "text/markdown" }
  },
  {
    "schema": "0.3",
    "name": "text-to-markdown",
    "since": "1.2.0",
    "description": "Converts plain text or HTML to clean formatted markdown",
    "tags": ["text", "conversion", "markdown"],
    "input": { "type": "stdin", "mediaType": "text/plain" },
    "output": { "type": "stdout", "mediaType": "text/markdown" },
    "params": [
      {
        "name": "style",
        "flag": "--style",
        "type": "enum",
        "enum": ["github", "plain", "obsidian"],
        "default": "github",
        "description": "Output markdown flavor"
      }
    ]
  }
]
```

A dispatcher encountering `deprecated: true` SHOULD:

- Exclude the tool from active tool listings by default
- Log a warning if it is invoked
- Automatically suggest `replacedBy` if present

---

### 3. Execution Contract

An OCT-compliant tool MUST follow these conventions:

1. **Input** arrives on `stdin` (if `input.type` is `stdin`)
2. **Output** is emitted on `stdout` (if `output.type` is `stdout`)
3. **Errors** are written to `stderr` only — never mixed into `stdout`
4. **Exit codes** follow Unix convention: `0` = success, non-zero = failure
5. **Stateless** — each invocation is independent. No assumption of prior runs.
6. **Flags** are passed as standard CLI arguments after the image name, following
   the tool's fixed `args`
7. **Sandbox-ready** — the tool MUST work under the baseline hardening flags in
   §7b, unless it declares an exception (e.g. `exec.network: true`)
8. **Bounded** — the tool SHOULD finish within the run time limit and SHOULD
   keep its output within the output limit (§7e)

#### Canonical Invocation Pattern

```bash
echo "$input" | docker run --rm -i <baseline flags (§7b)> \
  <image>@<digest> [args ...] [--flag value ...]
```

#### File Input and Output (Experimental)

> **Experimental:** File input and output are not stable in this draft. Field
> names and behavior may change, and dispatchers MAY decline to run tools that
> use them. Prefer `stdin` and `stdout` where possible.

File transfer is copy-in, copy-out. The dispatcher, not the tool, decides
which host data enters the container, and the tool never sees a host path.

When `input.type` is `file`:

- `input.path` is REQUIRED. It names the file inside the container and MUST be
  a single file name directly under `/in` (e.g. `/in/input.txt`).
- The dispatcher creates a new private temp directory, writes the input to it
  under that file name, and mounts the directory read-only at `/in`.
- If `input.flag` is present, the dispatcher passes it followed by
  `input.path` (e.g. `--input /in/input.txt`). Otherwise the tool is expected
  to read `input.path` without being told.

When `output.type` is `file`:

- `output.path` is REQUIRED. It MUST be a single file name directly under
  `/out` (e.g. `/out/result.md`).
- The dispatcher creates a new private temp directory, mounts it writable at
  `/out`, and after the run reads only `output.path` from it.
- If `output.flag` is present, the dispatcher passes it followed by
  `output.path` (e.g. `--output /out/result.md`).
- The output limit (§7e) applies to the file's size.

The dispatcher deletes both temp directories after every run, whether the run
succeeded or not. `path` and `flag` MUST NOT be present when `type` is not
`file`.

```bash
in="$(mktemp -d)"; out="$(mktemp -d)"
printf '%s' "$input" > "$in/input.txt"
docker run --rm <baseline flags (§7b)> \
  -v "$in":/in:ro -v "$out":/out \
  <image>@<digest> [args ...] --input /in/input.txt --output /out/result.md
cat "$out/result.md"; rm -rf "$in" "$out"
```

---

### 4. Discovery

#### 4a. Embedded — Local Image Scan

Scan all locally available images for the `com.teamropp.oct` label:

<!-- markdownlint-disable MD013 -->

```bash
docker images --format '{{.Repository}}:{{.Tag}}' \
  | xargs -I{} docker inspect --format \
    '{{if index .Config.Labels "com.teamropp.oct"}}{{.RepoTags}} {{index .Config.Labels "com.teamropp.oct"}}{{end}}' {} \
  | grep -v '^$'
```

<!-- markdownlint-enable MD013 -->

A dispatcher SHOULD cache this at startup and refresh when the local image list
changes. A dispatcher SHOULD record the image ID (digest) each label was read
from, so the image it later runs is the image it inspected (see §7d).

#### 4b. Embedded — Registry (Pre-Pull)

For images not yet pulled, the label is accessible via the registry API without
downloading layers:

```bash
# Using crane — no pull required
crane config <image>:<tag> | jq '.config.Labels["com.teamropp.oct"]'
```

#### 4c. External Manifest

An external manifest is a plain JSON file containing an array of **catalog
entries**. Each entry wraps a standard OCT tool array with the image reference
it applies to.

##### External Manifest Schema

```json
[
  {
    "image": "myrepo/text-tools",
    "tag": "1.2.0",
    "digest": "sha256:3f1c5e2a9b7d4c6e8f0a1b2c3d4e5f60718293a4b5c6d7e8f9a0b1c2d3e4f5a6",
    "source": "https://github.com/myrepo/text-tools",
    "tools": [
      {
        "schema": "0.3",
        "name": "text-to-markdown",
        "since": "1.2.0",
        "description": "...",
        "input": { "type": "stdin", "mediaType": "text/plain" },
        "output": { "type": "stdout", "mediaType": "text/markdown" }
      }
    ]
  },
  {
    "image": "someoneelse/aws-tools",
    "tag": "latest",
    "source": "https://hub.docker.com/r/someoneelse/aws-tools",
    "secrets": [
      {
        "name": "AWS_ACCESS_KEY_ID",
        "source": "env",
        "key": "AWS_ACCESS_KEY_ID",
        "required": true,
        "description": "AWS access key ID"
      },
      {
        "name": "AWS_SECRET_ACCESS_KEY",
        "source": "env",
        "key": "AWS_SECRET_ACCESS_KEY",
        "required": true,
        "description": "AWS secret access key"
      },
      {
        "name": "AWS_DEFAULT_REGION",
        "source": "env",
        "key": "AWS_DEFAULT_REGION",
        "required": false,
        "default": "us-east-1",
        "description": "AWS region"
      },
      {
        "name": "AWS_SESSION_TOKEN",
        "source": "env",
        "key": "AWS_SESSION_TOKEN",
        "required": false,
        "description": "Temporary session token"
      }
    ],
    "tools": [
      {
        "schema": "0.3",
        "name": "aws-whoami",
        "since": "latest",
        "description": "Returns current AWS caller identity",
        "input": { "type": "none" },
        "output": { "type": "stdout", "mediaType": "application/json" },
        "exec": { "network": true }
      },
      {
        "schema": "0.3",
        "name": "aws-s3-ls",
        "since": "latest",
        "description": "Lists S3 buckets or objects at a prefix",
        "input": { "type": "none" },
        "output": { "type": "stdout", "mediaType": "application/json" },
        "exec": { "network": true },
        "secrets": {
          "mode": "replace",
          "items": [
            {
              "name": "AWS_ACCESS_KEY_ID",
              "source": "env",
              "key": "AWS_ACCESS_KEY_ID",
              "required": true
            },
            {
              "name": "AWS_DEFAULT_REGION",
              "source": "env",
              "key": "AWS_DEFAULT_REGION",
              "required": false,
              "default": "us-east-1"
            }
          ]
        }
      }
    ]
  }
]
```

##### Catalog Entry Fields

| Field     | Required | Description                                                                                                   |
| --------- | -------- | ------------------------------------------------------------------------------------------------------------- |
| `image`   | Yes      | Image repository reference.                                                                                   |
| `tag`     | Yes      | Image tag the tools apply to. Human-readable; not used for integrity.                                         |
| `digest`  | No       | Content digest (`sha256:…`) of the image. When present, dispatchers MUST run `image@digest`, not `image:tag`. |
| `source`  | No       | URL of the image's source code or registry page.                                                              |
| `secrets` | No       | Entry-level secret items applied to every tool in the entry (see §5c).                                        |
| `tools`   | Yes      | Array of OCT tool objects (§2).                                                                               |

##### External Manifest Locations (Resolution Order)

A dispatcher SHOULD read the following sources. When more than one source
describes the same image, the source listed first wins: its catalog entry
replaces the others entirely, and fields are never merged across sources.
Images are matched by `digest` when both sources give one, otherwise by `image`
and `tag`. Images described by only one source are always included.

1. `~/.config/oct/catalog.json` — user's personal catalog
2. `./.config/oct/catalog.json` — project-local catalog (checked into source control)
3. Any URL registered in dispatcher config (e.g. a team-hosted catalog endpoint)
4. Embedded image labels (fallback)

##### Why External Manifests Matter

- **Describe images you don't own** — add OCT metadata to third-party images
  like `pandoc`, `ffmpeg`, `imagemagick` without forking them
- **Pre-pull discovery** — browse available tools before committing to a
  `docker pull`
- **Version correction** — fix a bad label in a published image without
  rebuilding it
- **Community catalogs** — a GitHub repo of `.json` files becomes a sharable,
  PR-reviewable tool registry with no infrastructure required
- **Air-gapped environments** — distribute a `catalog.json` alongside images in
  environments with no registry access

---

### 5. Secrets

Many tools require credentials or tokens to operate. OCT defines a structured
`secrets` field for declaring these requirements in a portable, backend-agnostic
way. The tool spec declares _what_ is needed; the user's own config decides
_where it comes from_.

A manifest (embedded label or external catalog) can only ask for secrets by
host environment variable name. Every other backend is opt-in by the user
through user-scope bindings (§5d). This prevents a manifest from directing a
dispatcher to read arbitrary files, keychain items, or vault paths.

#### 5a. Secret Item Schema

```json
{
  "name": "string (required)",
  "source": "env (required)",
  "key": "string (required)",
  "required": true,
  "default": "string",
  "description": "string"
}
```

| Field         | Description                                                                      |
| ------------- | -------------------------------------------------------------------------------- |
| `name`        | Environment variable name injected into the container (e.g. `AWS_ACCESS_KEY_ID`) |
| `source`      | MUST be `env` in a manifest. Other backends are set only via user bindings (§5d) |
| `key`         | Host environment variable to read the value from                                 |
| `required`    | If `true`, dispatcher MUST abort if the secret cannot be resolved                |
| `default`     | String value used when `required: false` and resolution fails                    |
| `description` | Human/LLM-readable description of what this secret is for                        |

Secret items MAY also carry configuration that is not sensitive, such as
`AWS_DEFAULT_REGION`, so that related values are declared and approved
together. Dispatchers MUST handle every item as a secret regardless. A later
version may add a separate field for configuration that is not sensitive.

A dispatcher MUST reject (treat as invalid and not invoke) any tool whose
manifest declares a secret `source` other than `env`.

#### 5b. Source Backends

The following backends are available **only in user-scope bindings**
(`~/.config/oct/config.json`, §5d). `env` is also the only backend a manifest
may declare.

| Source     | Resolution command                                                                          | `key` format                            |
| ---------- | ------------------------------------------------------------------------------------------- | --------------------------------------- |
| `env`      | Read from shell environment                                                                 | Env var name (e.g. `AWS_ACCESS_KEY_ID`) |
| `keychain` | macOS: `security find-generic-password -s KEY -w` / Linux: `secret-tool lookup service KEY` | Service name                            |
| `op`       | `op read KEY` (requires 1Password CLI)                                                      | `op://vault/item/field`                 |
| `vault`    | `vault kv get -field=value KEY` (requires HashiCorp Vault CLI)                              | `secret/path`                           |
| `file`     | Read file contents at path in `key`                                                         | Absolute file path                      |
| `prompt`   | Ask user interactively at invocation time; value is ephemeral                               | Display label shown to user             |

#### 5c. Inheritance and Merge Modes

Secrets are defined at two levels:

1. **Catalog entry level** — a `secrets` array on the catalog entry applies to
   **all tools** within that entry by default.
2. **Tool level** — a `secrets` object on an individual tool with a `mode` field
   controls how it merges with the entry-level secrets.

```json
"secrets": {
  "mode": "append | replace",
  "items": [ ... ]
}
```

| Mode               | Behavior                                                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `append` (default) | Tool's secrets are added to the entry-level secrets. Duplicate `name` values in the tool override the entry-level entry. |
| `replace`          | Tool's secrets fully replace the entry-level secrets. Use when a tool needs a narrower set than the entry default.       |

If a tool has no `secrets` field, it inherits the entry-level secrets unchanged.

An embedded label has no catalog entry, so there are no entry-level secrets. In
an embedded label, a tool's `secrets.items` are used as-is and `mode` has no
effect.

#### 5d. User Config Overrides

Users can override the `source` and `key` for any named secret in a config file
without modifying the catalog. This lets tool specs ship with `"source": "env"`
while users transparently redirect to their preferred backend.

```json
{
  "secrets": {
    "bindings": {
      "AWS_ACCESS_KEY_ID": { "source": "keychain", "key": "aws-access-key-id" },
      "AWS_SECRET_ACCESS_KEY": {
        "source": "keychain",
        "key": "aws-secret-access-key"
      },
      "AWS_SESSION_TOKEN": { "source": "keychain", "key": "aws-session-token" }
    }
  }
}
```

The same config file also holds tool rules (§6a) and run limits (§7e). A config
file's scope is set by where it lives, not by a field in the file:

- **User scope** — `~/.config/oct/config.json`. Applies to all projects, never
  committed. May bind any backend in §5b.
- **Project scope** — `.config/oct/config.json` in the project root. Can be
  shared with the team. Project bindings MUST use `source: "env"`; a dispatcher
  MUST ignore any other source in project scope, because project files arrive
  with untrusted repositories.

#### 5e. Resolution Algorithm

When a dispatcher prepares a tool invocation:

1. Collect the tool's effective secrets: entry-level secrets merged with
   tool-level secrets per the tool's `mode`.
2. If any manifest-declared secret uses a `source` other than `env`, reject the
   tool (§5a).
3. For each secret, look for a binding on that `name`, first in user scope
   (`~/.config/oct/config.json`), then in project scope
   (`.config/oct/config.json`). If found, use the first binding's `source` and
   `key` instead.
4. Before the first invocation of a tool that requests secrets, the dispatcher
   MUST show the user each secret's `name`, resolved `source`, and `key`, and
   obtain approval (see §7c).
5. Resolve the value using the appropriate backend command.
6. If `required: true` and resolution fails → abort with a clear error message
   naming the secret and source (never the value).
7. If `required: false` and resolution fails → use `default` if present,
   otherwise skip (do not inject this env var into the container).
8. Inject each resolved value by setting it in the environment of the `docker`
   client process the dispatcher spawns, and passing only `-e NAME` on the
   command line. Secret values MUST NOT appear in command-line arguments, where
   they are visible to other users through `ps` and process listings.
9. Resolved values MUST NOT be logged, written to disk, or included in error
   messages.

> **Note:** Environment variables of a running container remain visible to
> anyone with access to the Docker daemon (e.g. `docker inspect`) until the
> container exits. The `--rm` flag removes that record when the run ends.

---

### 6. Dispatcher Interface

An OCT dispatcher is the bridge between an OCT tool library and an LLM or agent.
It is responsible for:

1. **Discovery** — scanning local images, registry labels, and external
   manifests per §4
2. **Filtering** — excluding `deprecated` tools from active listings unless
   explicitly requested
3. **Translation** — converting OCT schemas into the tool definition format
   expected by the target LLM (Claude `tools` array, OpenAI function calling,
   etc.)
4. **Invocation** — building and executing `docker run --rm -i` with the
   baseline hardening flags (§7b), the tool's flags, and piped input when the
   LLM emits a tool call
5. **Return** — passing stdout to the LLM as the tool result; stderr + exit code
   on failure

#### 6a. Tool References and Name Collisions

A **tool reference** identifies one tool in one image:

```text
<repository>:<tag>#<tool>
myrepo/text-tools:1.2.0#text-to-markdown
ghcr.io/org/media-tools:2024-11#transcode
```

`<repository>` is written as Docker displays it. A **tool pattern** is a tool
reference in which `*` matches any run of characters, including none. All
other characters match literally, and a pattern must match the whole
reference:

| Pattern                   | Matches                                |
| ------------------------- | -------------------------------------- |
| `myrepo/text-tools:*#*`   | Every tool in every tag of one image   |
| `myrepo/*:*#convert`      | `convert` in any image under `myrepo/` |
| `myrepo/text-tools:1.*#*` | Every tool in the `1.x` tags           |
| `*#text-to-markdown`      | `text-to-markdown` in any image        |

The name a dispatcher shows to an LLM must be unique across all tools it
exposes. The dispatcher resolves duplicate tool names as follows:

1. **Same image** — the tool array is invalid (§2).
2. **Same repository, different tags** (e.g. `1.2.0` and `1.3.0` are both
   pulled) — the dispatcher exposes the tool from one image only. It SHOULD use
   the most recently created image unless user config disables the others.
3. **Different repositories** — the dispatcher MUST NOT expose either tool
   under the bare name. It exposes each as `<image>__<tool>`, where `<image>`
   is the last path segment of the repository (`text-tools__convert`). Tool
   names cannot contain `_`, so the `__` separator is unambiguous.
4. If a name is still not unique, or is longer than 64 characters, the
   dispatcher MUST NOT expose the affected tools until the user resolves the
   conflict in config.

Tools whose names are already unique keep their bare names. The dispatcher MUST
map the exposed name back to the full tool reference when the LLM calls it.

Users control naming in the `tools` array of the config file (§5d). Each rule
has a `match` pattern. Rules apply in order, and for each field the last
matching rule that sets it wins:

```json
{
  "tools": [
    { "match": "myrepo/text-tools:*#text-to-markdown", "name": "md" },
    { "match": "myrepo/text-tools:1.2.*#*", "enabled": false },
    { "match": "someoneelse/*:*#*", "enabled": false }
  ]
}
```

| Field     | Description                                                                 |
| --------- | --------------------------------------------------------------------------- |
| `match`   | Tool pattern (required).                                                    |
| `name`    | Name to expose to the LLM instead of the tool's own name. Same rules as §2. |
| `enabled` | `false` hides the matching tools from the LLM. Default: `true`.             |
| `limits`  | Overrides the run limits for matching tools (§7e).                          |

#### Claude Tool Translation Example

The dispatcher maps parameter types to JSON Schema types as follows: `string`
to `string`, `integer` to `integer`, `float` to `number`, `boolean` to
`boolean`, and `enum` to `string` with an `enum` list. When `input.type` is
`stdin` or `file`, the dispatcher adds a required `input` property for the
primary input. Fixed `args` never appear in the tool definition.

Given an OCT tool entry, the dispatcher generates:

```json
{
  "name": "text-to-markdown",
  "description": "Converts plain text or HTML to clean formatted markdown",
  "input_schema": {
    "type": "object",
    "properties": {
      "input": {
        "type": "string",
        "description": "The text or HTML content to convert"
      },
      "style": {
        "type": "string",
        "enum": ["github", "plain", "obsidian"],
        "default": "github",
        "description": "Output markdown flavor"
      }
    },
    "required": ["input"]
  }
}
```

When the LLM calls `text-to-markdown` with
`{ "input": "...", "style": "obsidian" }`:

<!-- markdownlint-disable MD013 -->

```bash
echo "$input" | docker run --rm -i --network none --read-only --cap-drop ALL \
  --security-opt no-new-privileges --memory 512m --cpus 1 --pids-limit 256 \
  myrepo/text-tools@sha256:3f1c5e2a9b7d4c6e8f0a1b2c3d4e5f60718293a4b5c6d7e8f9a0b1c2d3e4f5a6 --style obsidian
```

<!-- markdownlint-enable MD013 -->

---

### 7. Security Model

OCT metadata comes from image labels and catalogs that the user may not have
written. A dispatcher MUST treat every manifest field as untrusted input.

#### 7a. Dispatcher-Built Invocation

The dispatcher MUST construct the `docker run` command itself from the
structured fields of the tool object. It MUST NOT execute any command string,
shell fragment, or extra `docker run` option supplied by a manifest. Fixed
`args` and parameter values supplied by an LLM MUST be passed as separate
argument-vector entries, never interpolated into a shell string. String values
that start with `-` MUST be rejected (§2).

A manifest cannot request privileged options such as `--privileged`, host
mounts, `--pid host`, added capabilities, or devices. Tools that need these are
out of scope for OCT.

#### 7b. Baseline Hardening

Dispatchers SHOULD apply the following flags to every invocation by default:

| Flag                                 | Purpose                                                  |
| ------------------------------------ | -------------------------------------------------------- |
| `--network none`                     | No network unless `exec.network: true` is approved (§7c) |
| `--read-only`                        | Read-only root filesystem                                |
| `--tmpfs /tmp`                       | Writable scratch space that disappears with the run      |
| `--cap-drop ALL`                     | Drop all Linux capabilities                              |
| `--security-opt no-new-privileges`   | Block privilege escalation through setuid binaries       |
| `--memory`, `--cpus`, `--pids-limit` | Bound resource use; values are dispatcher configuration  |

Dispatchers MUST also enforce the run limits in §7e.

Users MAY loosen these defaults for a specific image in their user-scope config.
A manifest MUST NOT be able to loosen them, apart from requesting network access
through `exec.network`.

#### 7c. User Approval

A dispatcher MUST obtain explicit user approval before the first invocation of
a tool that requests network access (`exec.network: true`) or secrets. The
approval prompt MUST name the image (with digest when known), the secrets
requested, and whether network access is requested. Dispatchers SHOULD
remember approval per image digest and ask again when the digest changes.

Network access and secrets together allow a tool to send credentials off the
machine, so dispatchers SHOULD make that combination clear in the prompt.

#### 7d. Image Integrity

- When a catalog entry includes `digest`, the dispatcher MUST invoke the image
  as `image@digest`.
- When metadata comes from an embedded label, the dispatcher SHOULD invoke the
  image by the same digest it inspected, so a tag re-pushed after discovery
  cannot run with metadata the user approved for a different image.
- Dispatchers MAY offer a policy that refuses to run tools from catalog entries
  without a `digest`.

#### 7e. Run Limits

Every invocation runs under two limits:

| Limit       | Default                 | Description                                          |
| ----------- | ----------------------- | ---------------------------------------------------- |
| `timeout`   | `60` seconds            | Wall-clock time from container start to exit.        |
| `maxOutput` | `1048576` bytes (1 MiB) | Size of the result: stdout, or the output file (§3). |

When the timeout is reached, the dispatcher MUST stop the container itself
(e.g. by starting it with `--name` and running `docker rm -f`), because killing
the `docker` client process does not stop the container. When the output
exceeds `maxOutput`, the dispatcher MUST stop reading, stop the container, and
report an error. It MUST NOT pass truncated output to the LLM as if it were the
full result. In both cases the error names the limit that was reached.

Users set limits in the config file (§5d), globally or per tool through the
`tools` rules (§6a). A per-tool rule overrides the global value:

```json
{
  "limits": { "timeout": 60, "maxOutput": 1048576 },
  "tools": [
    {
      "match": "myrepo/video-tools:*#*",
      "limits": { "timeout": 600, "maxOutput": 10485760 }
    }
  ]
}
```

User-scope config MAY raise or lower limits. Project-scope config MAY only
lower them; a dispatcher MUST ignore a project-scope value that is higher than
the value that would otherwise apply.

#### 7f. Text Shown to the LLM

`description`, `tags`, `examples`, `exec.notes`, and the `description` fields
of inputs, outputs, parameters, and secrets are passed to the LLM as part of the
tool definition. They come from manifests the user may not have written and can
carry prompt injection: text that tries to make the model ignore its
instructions, call other tools, or leak data.

- Dispatchers MUST treat these fields as untrusted data and MUST NOT act on
  them as instructions themselves.
- A project-scope catalog (§4c) can replace the metadata of an image the user
  already uses, including these fields. Dispatchers SHOULD tell the user when a
  project-scope catalog overrides an image that another source also describes.
- Dispatchers SHOULD let the user review a tool's LLM-facing text when they
  approve it (§7c).

---

### 8. Dockerfile Example

```dockerfile
FROM python:3.12-slim

COPY convert.py /app/convert.py
RUN pip install --no-cache-dir mistune

ENTRYPOINT ["python", "/app/convert.py"]

LABEL com.teamropp.oct='[ \
  { \
    "schema": "0.3", \
    "name": "text-to-markdown", \
    "since": "1.2.0", \
    "description": "Converts plain text or HTML to clean formatted markdown", \
    "tags": ["text", "conversion", "markdown"], \
    "input": { "type": "stdin", "mediaType": "text/plain" }, \
    "output": { "type": "stdout", "mediaType": "text/markdown" }, \
    "params": [ \
      { \
        "name": "style", \
        "flag": "--style", \
        "type": "enum", \
        "enum": ["github", "plain", "obsidian"], \
        "default": "github", \
        "description": "Output markdown flavor" \
      } \
    ] \
  } \
]'
```

---

## Open Questions

1. **`since` field format** — Should `since` be a semver string, a date, or any
   arbitrary image tag? Freeform is flexible but hard to compare. A
   recommendation (not requirement) toward semver may be sufficient for v1.

2. **Community catalog hosting** — A GitHub repo of JSON files is the simplest
   viable catalog. Worth defining a standard repo structure and submission
   process early if community adoption is a goal.

3. **File input/output and in-place edits** — The experimental file variant
   (§3) copies data in and out through private temp directories. Should a later
   version also let a tool edit user files in place through a writable mount?
   That avoids copying large files, but it lets an untrusted tool change or
   delete host data and lets the LLM pick which host paths are exposed. One
   middle ground is a read-only bind mount for large inputs, with output still
   copied out for the dispatcher to apply.

4. **Streaming output** — Buffered stdout sufficient for v1? Streaming matters
   for large outputs (e.g. video processing).

5. **Multi-image tool chains** — Out of scope for v1, but worth noting: some
   workflows naturally chain tools. Should the external manifest support a
   `pipeline` concept?

6. **`prompt` source security** — When `source: "prompt"`, the user types a
   value into their terminal. Should the spec require that dispatchers zero the
   value after injection or mark the env var for post-run cleanup?

7. **Network scoping** — `exec.network` is all-or-nothing. Should a later
   version let tools declare the hosts they need, so dispatchers can enforce an
   egress allowlist?

8. **Manifest-declared limits** — Run limits (§7e) are set only by user config.
   Should a manifest be able to declare the time and output size a tool
   normally needs (e.g. a video tool), for the dispatcher to honor up to a
   user-set ceiling?

---

## Non-Goals (v1)

- Stateful tools / persistent processes (use MCP for those)
- Tools that serve network endpoints (HTTP servers, websockets). Tools may make
  outbound connections when they declare `exec.network: true`.
- Multi-step / chained tool definitions in a single invocation
- GUI or interactive tools
- Tools that need privileged containers, host mounts, or added capabilities
- Secret rotation, TTL, or auto-refresh hints (deferred to v2)
- Multi-user or team-shared secret vaults (use project scope in
  `.config/oct/config.json` for team defaults)

---

## Comparison to Related Work

| Project                | Model                    | Requires running server | Self-describing image | External catalog     | Secrets model                      |
| ---------------------- | ------------------------ | ----------------------- | --------------------- | -------------------- | ---------------------------------- |
| MCP                    | JSON-RPC over stdio/HTTP | Yes                     | No                    | Yes (MCP Registry)   | Out of band                        |
| Docker MCP Catalog     | MCP + registry metadata  | Yes                     | No                    | Yes (Docker-hosted)  | Out of band                        |
| labs-ai-tools-for-devs | Markdown + Docker + LLM  | Partial                 | No                    | No                   | None                               |
| OpenAPI plugins        | HTTP + spec URL          | Yes                     | No                    | Yes                  | None                               |
| **OCT (this spec)**    | stdin/stdout + label     | **No**                  | **Yes**               | **Yes (plain JSON)** | **Declared inline, bound by user** |

---

## License

Copyright 2026 Curtis Ropp / TeamRopp.

- The specification text is licensed under the
  [Creative Commons Attribution 4.0 International License](LICENSE-CC-BY-4.0.txt)
  (CC BY 4.0).
- Code, JSON schemas, and examples (including the code blocks in this document)
  are licensed under the [Apache License, Version 2.0](LICENSE-APACHE-2.0.txt).

---

Draft v0.3 — Curtis Ropp / TeamRopp — Open for review
