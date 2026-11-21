# CEP XXXX - Verifying Sigstore attestations of recipe sources

<table>
<tr><td> Title </td><td> Verifying Sigstore attestations of recipe sources </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Wolf Vollprecht &lt;wolf@prefix.dev&gt; </td></tr>
<tr><td> Created </td><td> May 20, 2026 </td></tr>
<tr><td> Updated </td><td> Nov 21, 2026 </td></tr>
<tr><td> Discussion </td><td> https://github.com/conda/ceps/pull/168 </td></tr>
<tr><td> Implementation </td><td> <a href="https://github.com/prefix-dev/rattler-build/pull/2947">prefix-dev/rattler-build#2947</a> </td></tr>
<tr><td> Requires </td><td> CEP 13, CEP 14 (recipe format) </td></tr>
</table>

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
  "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
  described in [RFC2119][RFC2119] when, and only when, they appear in all capitals, as shown here.
>

## Abstract

This CEP adds an `attestations` key to URL sources in the recipe format of [CEP 13] and [CEP 14].
It names the [Sigstore] attestations that a downloaded source archive must carry, and the
publishers that must have signed them. A build tool verifies those attestations before it uses the
source, and fails the build if verification fails.

The attestations are those that upstream projects already publish: PyPI provenance ([PEP 740]),
GitHub release attestations and build provenance, and any other Sigstore bundle. [CEP 27] and
[CEP 50] cover attestations of the conda packages a build *produces*. This CEP covers the sources a
build *consumes*, so that a package can carry a verified chain from the upstream repository to the
conda artifact.

## Motivation

A recipe pins a source by its checksum:

```yaml
source:
  url: https://files.pythonhosted.org/packages/source/f/flask/flask-3.1.1.tar.gz
  sha256: 6489f1...
```

The checksum proves that the archive is the one the recipe author downloaded. It does not prove
that this archive came from the upstream project. Recipe authors and automated update bots (for
example conda-forge's autotick bot) compute the checksum from whatever the URL returns at the
moment of the update. If an attacker has replaced the archive by then, the attacker's archive is
pinned. Such a replacement can come from:

- a compromised maintainer account on the package index,
- a mirror or CDN that serves modified content, or
- a release asset that someone with write access to the repository swapped out.

Upstream ecosystems now sign their releases with Sigstore:

- PyPI publishes [PEP 740] attestations for every file uploaded through Trusted Publishing, served
  through its [integrity API][PyPI Integrity].
- GitHub signs a release attestation for every [immutable release][GitHub immutable releases],
  covering the SHA-256 of each uploaded asset, and projects add SLSA build provenance with
  `actions/attest-build-provenance`.
- GitLab CI, npm and RubyGems publish similar provenance.

These attestations bind an archive to the repository, workflow and ref that produced it, and every
signature is recorded in a public transparency log. A recipe can state which publisher it expects.
The build tool can then refuse an archive that this publisher did not sign, even when the checksum
in the recipe was updated to match it.

## Specification

### The `attestations` key

A URL source (CEP 14, "URL source") MAY contain an `attestations` key. Its value is either one
*attestation entry* or a non-empty list of them. A single mapping is equivalent to a list with one
element.

```yaml
source:
  url: https://files.pythonhosted.org/packages/source/f/flask/flask-${{ version }}.tar.gz
  sha256: 284c7b8f2f58cb737f0cf1c30fd7eaf0ccfcde196099d24ecede3fc2005aa59e
  attestations:
    publishers:
      - github:pallets(16748505)/flask(596892)@refs/tags/${{ version }}
```

An empty list MUST be rejected as an error, so that an edit cannot turn verification off while
appearing to keep it. Other source types (`git`, `path`) do not take `attestations`; see
[Future work](#future-work).

An attestation entry is a mapping with these keys:

| Key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `bundle_url` | URL | no | Where to download the Sigstore bundle. When omitted, the build tool derives it ([Bundle discovery](#bundle-discovery)). |
| `publishers` | non-empty list of publishers | no | Publishers that must match the verified attestation. Every listed publisher MUST match. An empty list MUST be rejected as an error. |

Both keys MAY be templated with the Jinja syntax of CEP 13. An entry with neither key is an error.

Every entry is verified on its own, and every entry MUST verify. A recipe can therefore require
several independent kinds of evidence for one archive: for example, that a GitHub release asset is
covered by the immutable release attestation **and** by the build provenance of the release
workflow.

`attestations` does not replace the checksum. A recipe that uses `attestations` SHOULD still give
`sha256`. The checksum keeps the build reproducible, and the attestation shows that the pinned
bytes came from the expected publisher.

### Publishers

A publisher names a source-code repository on a forge that issues Sigstore identities. This CEP
defines the forges `github` (GitHub Actions and GitHub releases) and `gitlab` (GitLab CI on
gitlab.com). A publisher has a string form and a mapping form.

The string form is:

```text
<forge>:<owner>[(<owner_id>)]/<repository>[(<repository_id>)][@<ref>]
```

- `owner` and `repository` are matched case-insensitively.
- `owner_id` and `repository_id` are the forge's immutable numeric IDs of the owner (user,
  organization or group) and of the repository. Each is optional.
- `ref` is a full git ref such as `refs/tags/3.1.1` or `refs/heads/main`. It is optional.

The mapping form takes exactly one of `github` or `gitlab`, whose value is `owner/repository`
without IDs, and these optional keys: `ref`, `owner_id`, `repository_id` and `environment`.
`environment` is the deployment environment the signing workflow ran in (for example `release`).

```yaml
publishers:
  - github: pallets/flask
    ref: refs/tags/${{ version }}
    owner_id: 16748505
    repository_id: 596892
    environment: release
```

A publisher matches an attestation only if the repository and every value that the publisher pins
are present in the verified attestation and equal to it. A pinned value that the attestation does
not record does not match.

Recipe authors SHOULD pin `owner_id` and `repository_id`. A repository name can be reused: when an
account or repository is renamed or deleted, someone else can register the same name. The numeric
IDs are never reused.

### Bundle discovery

The build tool MUST locate the bundle for each entry in this order:

1. If `bundle_url` is given, download it.
2. If the source URL is a PyPI file (host `files.pythonhosted.org`, or host `pypi.org` or
   `pypi.io` with a path starting with `/packages/`), request the PyPI integrity API at
   `https://pypi.org/integrity/{project}/{version}/{filename}/provenance` and use its
   `attestation_bundles`. `{filename}` is the last path segment of the URL the file was finally
   downloaded from, after redirects. `{project}` and `{version}` are parsed from `{filename}`
   according to the PyPA [source distribution][sdist format] and [wheel][wheel format] file name
   specifications (for an sdist, the name and version are separated by the last `-` before the
   archive extension). `pypi.io` is a legacy alias of `pypi.org` that many existing recipes use.
3. If the source URL is a GitHub release asset
   (`https://github.com/{owner}/{repo}/releases/download/{tag}/{asset}`), query
   `https://api.github.com/repos/{owner}/{repo}/attestations/sha256:{digest}?predicate_type=release`,
   where `{digest}` is the SHA-256 of the downloaded file.
4. Otherwise, fail with an error asking for `bundle_url`.

The bundle MAY be a Sigstore bundle with a DSSE envelope containing an [in-toto] statement, or a
bundle with a plain message signature (as produced by `cosign sign-blob --bundle`). A response
that holds several bundles (a PyPI provenance object, or a GitHub attestations API listing)
verifies if at least one of its bundles verifies and matches all publishers of the entry.

The build tool MUST distinguish "no attestation exists" (for example HTTP 404 from the API) from
network errors, rate limits and server errors in its error messages. Both MUST fail the build.

A build tool MAY send credentials to the GitHub API (for example from `GITHUB_TOKEN`) to avoid its
rate limit. It MUST NOT send them to any other host.

### Verification

For each attestation entry, the build tool MUST perform all of the following against the
downloaded file. If any step fails, the build MUST fail.

1. **Trust root.** Load the Sigstore trust root through [TUF]: the Sigstore public-good root for
   workflow-signed attestations, or GitHub's root for GitHub release attestations.
2. **Certificate.** Verify the signing certificate chain against the trust root at a verified
   signing time.
3. **Signing time.** Verify the signing time through an inclusion proof in the transparency log,
   or for GitHub release attestations through an RFC 3161 timestamp from GitHub's timestamp
   authority.
4. **Signature.** Verify the signature over the DSSE payload or the message.
5. **Subject.** Verify that the SHA-256 digest of the downloaded file is a subject of the signed
   in-toto statement, or, for a message signature, that the signed digest equals it.
6. **Publishers.** Verify that every listed publisher matches, as described below.

If `publishers` is omitted, steps 1–5 still apply, any signer is accepted, and the build tool
SHOULD print a warning. An empty `publishers` list is an error (see above), so that an edit cannot
drop the identity check while appearing to keep it.

#### Workflow-signed attestations

This applies to PyPI provenance, build provenance and other bundles signed by a CI workflow.

- The certificate's OIDC issuer MUST be `https://token.actions.githubusercontent.com` for `github`
  and `https://gitlab.com` for `gitlab`.
- The certificate's Subject Alternative Name (SAN) identifies the workflow, for example
  `https://github.com/pallets/flask/.github/workflows/publish.yml@refs/tags/3.1.1`. The repository
  is the path that follows the forge's host.
- The ref is the part that follows the **first** `@` after the repository path. Git ref names may
  contain `@`. Splitting at the last `@` would let a branch named `x@refs/tags/3.1.1` pass as the
  tag `3.1.1`.
- `owner_id` is compared with the Fulcio *Source Repository Owner Identifier* extension
  (OID `1.3.6.1.4.1.57264.1.17`) and `repository_id` with the *Source Repository Identifier*
  extension (OID `1.3.6.1.4.1.57264.1.15`).
- These extensions describe the repository that *triggered* the workflow, while the SAN names the
  repository that *contains* the workflow. The two differ when a project signs through a reusable
  workflow from another repository. The build tool MUST check that the *Source Repository URI*
  extension (OID `1.3.6.1.4.1.57264.1.12`) names the same repository as the SAN, and fail
  otherwise, so that the repository, the IDs and the ref of a publisher are always taken from the
  same repository. See [open question 2](#open-questions).
- Fulcio certificates do not record the deployment environment of the workflow run.
  `environment` can therefore only be checked against the `publisher` object of a PyPI provenance
  response, which PyPI fills from the OIDC token it verified at upload. This value is asserted by
  PyPI and is not covered by the Sigstore signature. For any bundle not discovered through the
  PyPI integrity API, a publisher that pins `environment` never matches.

The ref in a workflow certificate is the ref of the signing *workflow run*. A project that
publishes from a `workflow_dispatch` run on `main` is signed with `refs/heads/main`, not with the
release tag.

#### GitHub release attestations

A GitHub release attestation is signed by GitHub's release service, not by a workflow.

- Its certificate identity MUST be exactly `https://dotcom.releases.github.com`.
- The repository, `owner_id` and `repository_id` are taken from the signed in-toto statement of
  the same bundle, whose subjects include `pkg:github/{owner}/{repo}@{tag}`.
- A `ref` MUST be a tag ref (`refs/tags/{tag}`) and is compared with the release tag.
- A release attestation records no deployment environment, so a publisher that pins `environment`
  never matches one.

GitHub's generated "Source code" archives (`/archive/...`) are not covered by a release
attestation: the attestation records the commit, not the bytes of the generated archive.
Projects that want their source to be verifiable SHOULD upload a source archive as a release
asset.

### Mirrors

When a URL source lists several URLs, the build tool derives the bundle for the URL it actually
downloaded from. If no bundle can be derived for that URL and no `bundle_url` is given,
verification fails. Since the digest is what is verified, a `bundle_url` works for every mirror.

### Recording the verification

A build tool SHOULD record the verified attestations in the recipe metadata it stores in the
package (rattler-build uses `info/recipe/rendered_recipe.yaml`), including, for each publisher, the `owner_id` and
`repository_id` found in the attestation even when the recipe did not pin them. A consumer can then
see which repository vouched for the source of a package.

### Caching

A build tool that caches downloaded sources MUST NOT skip verification because a cached copy
exists. It MUST verify again whenever the `attestations` configuration of the source changes. It
MAY cache the result of a successful verification for an unchanged configuration and an unchanged
file digest.

## Examples

A PyPI source, pinned to the release tag and to immutable IDs:

```yaml
source:
  url: https://files.pythonhosted.org/packages/source/f/flask/flask-${{ version }}.tar.gz
  sha256: 284c7b8f2f58cb737f0cf1c30fd7eaf0ccfcde196099d24ecede3fc2005aa59e
  attestations:
    publishers:
      - github:pallets(16748505)/flask(596892)@refs/tags/${{ version }}
```

A GitHub release asset that must be both part of the immutable release and built by the release
workflow:

```yaml
source:
  url: https://github.com/astral-sh/uv/releases/download/${{ version }}/source.tar.gz
  sha256: ${{ sha256 }}
  attestations:
    # The immutable release attestation, discovered automatically
    - publishers:
        - github:astral-sh(115962839)/uv(699532645)@refs/tags/${{ version }}
    # The SLSA build provenance of the release workflow
    - bundle_url: https://api.github.com/repos/astral-sh/uv/attestations/sha256:${{ sha256 }}?predicate_type=provenance
      publishers:
        - github:astral-sh(115962839)/uv(699532645)@refs/heads/main
```

A bundle published next to the archive:

```yaml
source:
  url: https://example.org/releases/foo-${{ version }}.tar.gz
  sha256: ...
  attestations:
    bundle_url: https://example.org/releases/foo-${{ version }}.tar.gz.sigstore.json
    publishers:
      - github: example/foo
        ref: refs/tags/v${{ version }}
        owner_id: 1234567
        repository_id: 7654321
```

## Rationale

**Why verify in the build tool rather than in a bot?** An update bot that checks attestations
protects only the updates it makes. Verifying at build time protects every build of the recipe:
local builds, rebuilds of old recipes, and recipes edited by hand.

**Why exact identities instead of regular expressions?** Tools such as `cosign` accept identity
regular expressions, and those are easy to get wrong (an unanchored `github.com/org/.*` also matches
`github.com/org/repo-fork`). The structured publisher is parsed and compared field by field.

**Why a list of entries?** Different attestations prove different things. A release attestation
proves that the file belongs to the immutable release at a tag. Build provenance proves which
workflow and ref built it. A ref also means something different in each. Separate entries, each
with its own publishers, keep these apart.

**Why fail closed?** `attestations` in a recipe states that unverified sources are not acceptable.
If the build silently continued when the bundle could not be fetched, an attacker who can block
that request could disable verification.

## Security considerations

- **Name reuse:** pinning `owner_id` and `repository_id` protects against a renamed or deleted
  repository whose name is registered again. Build tools SHOULD tell the user the IDs of a
  successful verification, so that they can pin them.
- **Ref parsing:** see the rule about the first `@` above.
- **Publishers omitted:** an entry without publishers proves only that *someone* signed the file
  with a Sigstore identity. That has little value, so build tools warn about it.
- **Scope of the guarantee:** a valid attestation shows that the archive was produced by the named
  repository's CI. It does not show that the repository's code is trustworthy, and it does not
  protect against a compromised workflow in that repository. Pinning a tag `ref` (and, for PyPI
  sources, `environment`) narrows what an attacker with push access can do.
- **Availability:** verification needs network access to the bundle source and to the TUF
  repository. Offline or air-gapped builds need the bundles and trust root mirrored.

## Backwards compatibility

The `attestations` key is new. A build tool that does not implement this CEP SHOULD reject a recipe
that uses it rather than ignore the key, because ignoring it would build the source without the
verification the recipe asks for. Build tools that validate recipes against the recipe JSON schema
already reject unknown keys.

rattler-build implemented an experimental precursor from version 0.65.0 to 0.80.2, under the key
`attestation` (singular, one mapping) and behind its `--experimental` flag. Build tools MAY accept that key in rendered recipes, so that packages built
with those versions can still be rebuilt with verification. Recipes MUST use `attestations`.

## Open questions

1. **Attestation types:** should verification require a specific predicate type for each
   discovery path (the PyPI publish attestation for PyPI, SLSA provenance for build provenance),
   or is any signed statement that names the digest acceptable, as now?
2. **Reusable workflows:** the repository is matched against the workflow identity in the
   certificate SAN. When a project signs through a reusable workflow from another repository, the
   SAN names that other repository. Should the publisher be compared with the Fulcio
   source-repository extensions instead, and the ref with the source-ref extension rather than with
   the SAN? This would lift the requirement that both name the same repository.
3. **GitHub release without publishers:** should a GitHub release asset without `publishers`
   default to the repository in the download URL, rather than accepting any release attestation for
   the digest?
4. **More forges:** should this CEP define publisher prefixes for other OIDC issuers (Buildkite,
   CircleCI, self-hosted GitHub and GitLab instances, Google Cloud service accounts whose identity
   is an email address), or leave them to a later CEP?
5. **GitLab subgroups:** GitLab projects can live in nested groups (`group/subgroup/project`).
   Should the string form treat everything before the last `/` as the owner, or should GitLab
   publishers use only the mapping form?
6. **Environment:** `environment` can only be checked for PyPI, and only against a value asserted
   by PyPI rather than signed. Is that enough to keep the key, or should it be removed?
7. **Policy outside the recipe:** should channels (for example conda-forge) be able to require
   attestations for sources from PyPI or GitHub releases, through a configuration of the build tool,
   when the recipe does not ask for them?

## Future work

- `git` sources: verify signed tags or commits (Sigstore `gitsign`, SSH or GPG signatures).
- Linking the verified source attestation into the package's publish attestation ([CEP 27]), so
  that a consumer can follow the chain from the conda package back to the upstream repository.

## References

- [Sigstore]
- [Sigstore bundle format][Sigstore Bundle]
- [in-toto]
- [PEP 740]
- [PyPI integrity API][PyPI Integrity]
- [GitHub immutable releases][GitHub immutable releases]
- [CEP 27] and [CEP 50]

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

[RFC2119]: https://www.ietf.org/rfc/rfc2119.txt
[Sigstore]: https://sigstore.dev
[Sigstore Bundle]: https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_bundle.proto
[in-toto]: https://in-toto.io
[TUF]: https://theupdateframework.io
[PEP 740]: https://peps.python.org/pep-0740/
[sdist format]: https://packaging.python.org/en/latest/specifications/source-distribution-format/
[wheel format]: https://packaging.python.org/en/latest/specifications/binary-distribution-format/
[PyPI Integrity]: https://docs.pypi.org/api/integrity/
[GitHub immutable releases]: https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases
[CEP 13]: ./cep-0013.md
[CEP 14]: ./cep-0014.md
[CEP 27]: ./cep-0027.md
[CEP 50]: ./cep-0050.md
