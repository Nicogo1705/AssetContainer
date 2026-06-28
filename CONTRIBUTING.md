# Contributing an asset

## 1. Prepare your repository

Your asset lives in **your own public git repository** (GitHub or any host). At its root:

```
AssetData/
├── manifest.json     # required — see schemas/manifest.schema.json
├── README.md         # long description (markdown)
├── thumbnail.png     # thumbnail ~512x512
├── screenshots/      # optional captures
└── <your Stride project: .csproj, .sd*, .cs, resources…>
```

`manifest.json` rules:
- Unique reverse-DNS `id`, e.g. `com.yourhandle.super-asset` (= registry file name).
- `category` ∈ `catalog/categories.json`, `license` ∈ `catalog/licenses.json` (SPDX).
- **Do not set `strideVersion`** unless detection fails: it is read from the `.csproj`.
- **`dependencies`**: let the publishing tool infer them from your `<ProjectReference>` entries,
  or list the `id`s of the other store assets you require.

## 2. Submit the entry

Open a PR adding `registry/<your-id>.json` (see `schemas/registry-entry.schema.json`):

```json
{
  "id": "com.yourhandle.super-asset",
  "repo": "https://github.com/yourhandle/SuperAsset",
  "submittedBy": "yourhandle",
  "latest": { "ref": "main" }
}
```

The `commit` field is resolved automatically by the bot — leave it empty or zeroed.

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

Every asset can always be imported as **source** (`localImport`): the app clones the repo and adds a
`<ProjectReference>`, so users can compile and modify it locally.

If you also publish your asset on **NuGet**, declare it in the manifest to enable `nugetImport`
(the app adds a `<PackageReference>` instead of cloning):

```json
"defaultImport": "nuget",
"nuget": { "packageId": "YourName.YourAsset", "packageVersion": "1.0.0" }
```

`defaultImport` only suggests the preferred mode; users can still choose source import.

## Good to know

- Only **source + `.sd*` + resources** are cloned (no binaries). Keep `AssetData/` light; use
  Git LFS / releases for large files.
- Content may include code (`.cs`): it is surfaced to the user, never executed automatically.
