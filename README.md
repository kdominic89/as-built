# as-built-dotnet

Keeps the structural part of a .NET repository's architecture document in step
with its code.
One program in a container, with the C# helper that reads the sources and the
.NET runtime the helper runs on: no network.

## Use

Restore the repository, then run the image in it:

```sh
docker run --rm --network none \
  --volume "$PWD:$PWD:ro" \
  --volume "$HOME/.nuget/packages:$HOME/.nuget/packages:ro" \
  --workdir "$PWD" \
  ghcr.io/kdominic89/as-built-dotnet check docs/architecture.md
```

Every finding names its file and line.
Any finding is a non-zero exit, so the same line works as a pull request check.

The two mounts are at the paths they have on your machine on purpose;
[Restore first](#restore-first) says why,
and how a pull request check does it.

## Commands

| Command | What it does |
| --- | --- |
| `check [FILE]` | Says whether the diagrams and tables the program owns in `FILE` match the code. Writes nothing. |
| `write [FILE]` | Writes those diagrams and tables again from the code. |

`FILE` defaults to `docs/architecture.md`.
`--root` names the directory the code is read from where it is not the current
one.

`write` changes the file, so it needs the mount writable.
On Linux it also needs your own identity,
because the program in the image does not run as you:

```sh
docker run --rm --network none --user "$(id -u):$(id -g)" \
  --volume "$PWD:$PWD" \
  --volume "$HOME/.nuget/packages:$HOME/.nuget/packages:ro" \
  --workdir "$PWD" \
  ghcr.io/kdominic89/as-built-dotnet write docs/architecture.md
```

Text you wrote around the diagrams and tables is kept as it is.

## What the program owns

The program owns a block by its name and nothing else in the file.
A block it owns is written from the code; a block that is not there is not
added, so you decide what the document shows by putting the empty block where
you want it and running `write`.

A Mermaid diagram is owned by its `accTitle`:

| `accTitle` | What it shows |
| --- | --- |
| `Package dependencies` | The projects of the tree and what references what. |
| `System context` | The shipped projects and the packages they take from a registry. |
| `Areas of <project>` | The directories of one project and which uses which. |
| `Verification` | The projects that verify the shipped ones. |
| `Reach of <Type.Member>` | What a call of that member reaches. |

An empty block is enough to start one:

````markdown
```mermaid
  accTitle: Package dependencies
```
````

A table is owned by its header row, under a heading that names a project:
`| Area | Uses |` and `| Area | Decisions |`.
The second lists the decision records under `docs/decisions` that link into
an area from their `Implementation References` section;
`--decisions` names another directory.

`--shipped` names which projects are the shipped ones, as a glob of their
project files; it defaults to `src/**`.

Every other diagram, table and paragraph is yours and is left as it is.

## What else is checked

- A list item of the form `` `A` does not use `B` `` is held against the code,
  with `A` and `B` naming projects or areas; a use is a finding that names the
  lines that use it.
- A link to a file or a heading in the repository has to lead somewhere.
- A Mermaid diagram carries an `accTitle` and an `accDescr`.
- The document names no absolute path of a machine.
- The document holds only ASCII; a diagram or a table row may also hold
  box-drawing characters, arrows, shapes and the three status marks.

## Restore first

The program reads the sources as a build does,
and a call into a NuGet package is only told from any other name once the
package's assemblies can be read.
`dotnet restore` writes where they are into `obj/project.assets.json` of each
project, as absolute paths of the machine that restored.
Those paths have to exist in the container.
Where they do not, nothing fails:
the calls into packages are left unbound and the document is held to less than
the code says.

A tree restored on your machine names your package folder,
`$HOME/.nuget/packages` unless you moved it.
Mount the repository and that folder at their own paths, as above.
This works on Linux and macOS;
a tree restored on Windows names paths no container has.

In a pull request check, or wherever the tree was not restored on the machine
itself, restore in a container with the package folder under the mount,
then mount the repository at the same place:

```sh
docker run --rm --user "$(id -u):$(id -g)" \
  --volume "$PWD:/work" --workdir /work \
  --env HOME=/tmp --env NUGET_PACKAGES=/work/.nuget/packages \
  mcr.microsoft.com/dotnet/sdk:<the version your build uses> \
  dotnet restore

docker run --rm --network none --volume "$PWD:/work:ro" \
  ghcr.io/kdominic89/as-built-dotnet check docs/architecture.md
```

```sh
docker run --rm --network none --user "$(id -u):$(id -g)" \
  --volume "$PWD:/work" \
  ghcr.io/kdominic89/as-built-dotnet write docs/architecture.md
```

That restore leaves `.nuget/packages` in the repository,
so name it in `.gitignore`.
It also writes the `obj` directories again with the container's paths:
on a machine you build on, restore once more afterwards.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | No finding. |
| 1 | At least one finding; each is printed as `path:line: message`. |
| 2 | The tree could not be judged, for example a directory that holds no architecture document, or a mount that is read-only when the document is to be written. |

## Pin it

A tag can be moved and a digest cannot.
In a workflow, name the image by the digest of the version you checked,
`ghcr.io/kdominic89/as-built-dotnet@sha256:...`, and not by its tag.
The digest of every version is in the [changelog](CHANGELOG.md).

## Reporting a problem

A finding you think is wrong, or a document the program should refuse and does
not: open an issue here, with the command and the smallest tree that shows it.
A vulnerability is reported privately, as [SECURITY.md](SECURITY.md) says.

The source is not published, so there is nothing to send a change against.

## Terms

Free to run and to pass on unmodified, also commercially.
The full terms are in [TERMS.md](TERMS.md).
The same file is in the image under `/licenses`,
beside the license notices of the open-source crates linked into the program
and the license texts of the .NET runtime and the compiler assemblies that ship
with it.
