# pnpm sbom: invalid URL in CycloneDX BOM `externalReferences`

`pnpm sbom` emits a non-URL string (npm `org/repo` repository shorthand) into
`components[].externalReferences[].url`, producing a BOM that is invalid per
the CycloneDX schema and is rejected by servers such as Dependency-Track with
HTTP 400 "Schema validation failed".

## Environment

- pnpm 12.3.4 (`packageManager` field, `pnpm-workspace` not involved)
- CycloneDX spec version 1.6
- Node.js 24.x (darwin-arm64)

## Reproduction

```sh
pnpm install          # installs ms@2.1.3, the only dependency
pnpm sbom --sbom-format cyclonedx --sbom-spec-version 1.6 --out pnpm-bom.json
```

`ms@2.1.3` declares its repository using the npm shorthand, not a URL:

```json
// node_modules/ms/package.json
{
  "name": "ms",
  "version": "2.1.3",
  "repository": "vercel/ms"
}
```

## Actual result

`pnpm-bom.json` (committed in this repo) contains:

```json
// pnpm-bom.json -> components[0] (ms@2.1.3) -> externalReferences[1]
{
  "type": "vcs",
  "url": "vercel/ms"
}
```

`"vercel/ms"` is not a valid RFC 3986/3987 IRI-reference (no scheme, no
authority), so the BOM fails schema validation.

Submitting the BOM to Dependency-Track fails:

```
[DependencyTrack] Reading artifact "bom.json"
[DependencyTrack] Publishing artifact "bom.json" to Dependency-Track
[DependencyTrack] Invalid payload submitted to server
{"status":400,"title":"The uploaded BOM is invalid","detail":"Schema validation failed","errors":[
  "$.components[0].externalReferences[1].url: does not match the iri-reference pattern must be a valid RFC 3987 IRI-reference",
  "$.components[0].externalReferences[1].url: does not match the regex pattern ^urn:cdx:[0-9a-f]{8}-...-...-...-[0-9a-f]{12}/[1-9][0-9]*$#..."
]}
```

(We originally observed this in a much larger project where dozens of
components produced the same `does not match the iri-reference pattern`
errors for `externalReferences[].url`; each of those components has an
`org/repo`-shorthand `repository` field in its `package.json`.)

## Expected result

The `url` of an `externalReference` must be a valid URI. pnpm should either:

1. normalize shorthand repositories to a full URL, e.g.
   `vercel/ms` → `https://github.com/vercel/ms` (like npm does), or
2. omit the `externalReference` when the `repository` value is not a valid URL.

For comparison, OWASP cdxgen 12.8.4 (`pnpm dlx @cyclonedx/cdxgen -t
typescript -o cyclonedx-cdxgen-bom.json --spec-version 1.6`) simply does not
emit a `vcs` external reference for `ms` at all, and its BOM is accepted by
Dependency-Track. A reference BOM generated this way is committed in this repo
as `cyclonedx-cdxgen-bom.json`.

## Related

- [pnpm/pnpm#11599](https://github.com/pnpm/pnpm/issues/11599) — same class of bug, but for the
  project's *own* repository: an SSH-style `repository` value
  (`git@some.host/some-repo.git`) is written verbatim into
  `$.metadata.component.externalReferences[0].url` and rejected by
  Dependency-Track with the same iri-reference error. Closed as **not
  planned**. This issue is a distinct instance: invalid values in
  *dependency* components' `externalReferences`, caused by npm's very common
  `org/repo` shorthand for `repository`.

## Files

| File | Description |
| --- | --- |
| `package.json` | Minimal project, single dependency `ms@^2.1.3` |
| `pnpm-lock.yaml` | Lockfile from `pnpm install` (pnpm 12.3.4) |
| `pnpm-bom.json` | BOM produced by `pnpm sbom` (invalid URL at `components[0].externalReferences[1]`) |
| `cyclonedx-cdxgen-bom.json` | Reference BOM produced by cdxgen for comparison (valid) |
