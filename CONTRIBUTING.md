# Contributing an asset

## 1. Prepare your repository

Your asset lives in **your own public git repository** (GitHub or any host). At its root:

```
your-repo/
├── README.md         # long description (markdown) — repo root, NOT cloned
├── thumbnail.png     # ~512x512 cover — repo root, NOT cloned
├── media/            # optional gallery images / short videos — repo root, NOT cloned
├── LICENSE.md        # repo root
└── AssetData/        # the ONLY folder cloned into a project (sparse checkout)
    ├── manifest.json   # required — see schemas/manifest.schema.json
    └── <your Stride project: .csproj, .sd*, .cs, resources…>
```

`thumbnail` and `media` paths in the manifest are relative to the **repository root** (they're
display-only, so they aren't cloned into projects; for a pack whose media *is* its content — e.g.
textures — point at `AssetData/…`).

`manifest.json` rules:
- Unique reverse-DNS `id`, e.g. `com.yourhandle.super-asset` (= registry file name).
- `category` ∈ `catalog/categories.json`, `license` ∈ `catalog/licenses.json` (SPDX).
- **Do not set `strideVersion`** unless detection fails: it is read from the `.csproj`.

**Your `.csproj` must reference a Stride version that exists on nuget.org.** This is the single most
common way to publish an asset nobody can use: a version built locally from the Stride sources
resolves perfectly on your machine and fails with `NU1102` on everyone else's. Check before you
submit:

```bash
dotnet restore --source https://api.nuget.org/v3/index.json -p:RestorePackagesPath=./.check
```

At the time of writing the newest published build is `Stride.Engine 4.4.0-beta5`.

**The asset compiler was renamed in 4.4**: `Stride.Core.Assets.CompilerApp` became
`Stride.AssetCompiler`, and only the new name has 4.4 releases. Keeping the old package next to a
4.4 engine looks fine while you build the library on its own — and then breaks every game that
references it, because the 4.3 compiler cannot load 4.4 assemblies. Reference `Stride.AssetCompiler`
at the same version as the engine.

Users on another Stride build can retarget on the fly with
`strideassetstore add <id> --stride <version>`.
- **`dependencies`**: let the publishing tool infer them from your `<ProjectReference>` entries,
  or list the `id`s of the other store assets you require.

## 2. Submit the entry

Open a PR adding `registry/<your-id>.json` (see `schemas/registry-entry.schema.json`):

```json
{
  "id": "com.yourhandle.super-asset",
  "repo": "https://github.com/yourhandle/SuperAsset",
  "latest": { "ref": "main" }
}
```

Leave `commit` out. The commit `ref` points at is resolved when the index is built and recorded in
`index.lock.json`; nothing ever writes it back into your `registry/<id>.json`.

## 3. Automated validation

When the PR opens, CI will:
1. validate `registry/<id>.json` and `AssetData/manifest.json` against the schemas;
2. clone your repo at `latest.ref`, resolve and **pin the commit**;
3. compute the **SHA-256 hash** of `AssetData/`;
4. detect the **Stride version** from the `.csproj`;
5. check that `dependencies` ⊇ `<ProjectReference>` pointing at store assets;
6. comment the result (✅ / ⚠️ / ❌).

On merge, `index.lock.json` is regenerated.

## 4. Certification (optional)

A "certified" badge is granted **by the registry maintainers** (CODEOWNERS): it adds an entry to
`certified[]` pinning a specific reviewed commit. You cannot self-certify. A certified asset must
only depend on certified assets. (This is the registry's own review — it is **not** an endorsement
by the Stride project.)

## Import modes (local vs NuGet)

Every asset can always be imported as **source** (`defaultImport: "local"`): the app clones the repo
and adds a `<ProjectReference>`, so users can compile and modify it locally.

If you also publish your asset on **NuGet**, declare it in the manifest and set `defaultImport: "nuget"`
(the app adds a `<PackageReference>` instead of cloning):

```json
"defaultImport": "nuget",
"nuget": { "packageId": "YourName.YourAsset", "packageVersion": "1.0.0" }
```

`defaultImport` only suggests the preferred mode; users can still choose source import.

## Good to know

- Inside `AssetData/`, **everything you commit** is brought over — no file-type filtering. Nothing
  outside it is: the store checks out that folder only. By
  convention, commit **source + `.sd*` + resources** only — not build output (`bin/`, `obj/`, `.dll`).
  Keep `AssetData/` light; use Git LFS / releases for large files.
- Content may include code (`.cs`): it is surfaced to the user, never executed automatically.
