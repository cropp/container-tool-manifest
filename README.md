# Container Tool Manifest (OCT)

OCT (Open Context Tool) is a small convention for describing a command-line tool
inside its container image, so AI agents and other dispatchers can find it and
run it with a plain `docker run`. There is no server, protocol handshake, or
registry to set up.

> **Status: Draft v0.3, published for review only.** This is not a released
> standard. I have been using OCT for a while with a skill of my own that builds
> OCT tools from existing command-line tools and runs them. I'm sharing the spec
> to get feedback on the design. Field names and rules may change before 1.0.

## How it works

An image describes its tools in the `com.teamropp.oct` label:

```dockerfile
LABEL com.teamropp.oct='[{
  "schema": "0.3",
  "name": "text-to-markdown",
  "description": "Converts plain text or HTML to clean formatted markdown",
  "since": "1.2.0",
  "input": { "type": "stdin", "mediaType": "text/plain" },
  "output": { "type": "stdout", "mediaType": "text/markdown" },
  "params": [{ "name": "style", "flag": "--style", "type": "enum",
               "enum": ["github", "plain", "obsidian"] }]
}]'
```

A dispatcher reads the label with `docker inspect`, or from the registry without
pulling, turns each tool into a tool definition for an LLM, and runs it in a
locked-down container when the model calls it:

```bash
echo "$input" | docker run --rm -i --network none --read-only --cap-drop ALL \
  myrepo/text-tools@sha256:<digest> --style obsidian
```

You can also describe images you don't own in an external `catalog.json`.

## Repository contents

| Path                                                                       | What it is                                   |
| -------------------------------------------------------------------------- | -------------------------------------------- |
| [`container-tool-manifest-spec.md`](container-tool-manifest-spec.md)       | The specification                            |
| [`schema/oct-0.3.schema.json`](schema/oct-0.3.schema.json)                 | JSON Schema for the image label (tool array) |
| [`schema/oct-catalog-0.3.schema.json`](schema/oct-catalog-0.3.schema.json) | JSON Schema for external catalog files       |

## Validating a manifest

Any JSON Schema draft 2020-12 validator works. With
[check-jsonschema](https://github.com/python-jsonschema/check-jsonschema):

```bash
check-jsonschema --schemafile schema/oct-0.3.schema.json my-label.json
check-jsonschema --schemafile schema/oct-catalog-0.3.schema.json catalog.json
```

The catalog schema refers to the label schema by relative path, so keep the two
files in the same directory.

## Feedback

Open an issue for questions, problems, or answers to the spec's
[Open Questions](container-tool-manifest-spec.md#open-questions).

## License

- Specification text: [CC BY 4.0](LICENSE-CC-BY-4.0.txt)
- Schemas, code, and examples: [Apache 2.0](LICENSE-APACHE-2.0.txt)
