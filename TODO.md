# TODO

Fork-local notes, kept on this branch rather than on `main` so the fork's
`main` stays identical to `upstream/main`. Not part of any upstream pull
request: branch from `upstream/main` when opening one.

## HCP Packer: report the source image so ancestry is linked

Status: not implemented. Follow-up to the registry metadata support added in
504fef9.

`Artifact.State()` now returns image ID, region and provider, so a build with an
`hcp_packer_registry` block publishes and reaches `BUILD_DONE`. It never calls
`registryimage.WithSourceID`, so the build record always stores an empty
`source_external_identifier`. HCP Packer therefore shows no parent for the
version: `Version.parents` is null and `has_descendants` is false.

Confirmed against the live HCP Packer API for a build from this branch:

```
build   component_type=qemu.rhel9  platform=transcend.qemu  status=BUILD_DONE
        labels={...}  source_external_identifier=""
artifact external_identifier=hcp-test.qcow2  region=output-hcp-test
version parents=null  has_descendants=false
```

That build is otherwise complete: `external_identifier` and `region` are the only
artifact fields a builder can supply, and everything else in the payload
(`metadata` with Packer/OS/plugin/CI/VCS details, `packer_run_uuid`, status,
timestamps) is supplied by Packer core, with `labels` coming from the template's
`hcp_packer_registry { build_labels = ... }`.

Implementation sketch:

- When `disk_image = true` (the build converts an existing disk image rather than
  installing from an ISO), record the source image and pass its identifier to
  `registryimage.WithSourceID(...)`. Deriving it the same way as the artifact ID
  (base name of the source file) keeps parent and child identifiers consistent.
- Leave it empty for ISO installs: an ISO is not a registry image and there is
  nothing to link.
- Precedent in other builders: `packer-plugin-amazon` uses `SourceAMI`,
  `googlecompute` and `azure` use `SourceImageName`.

Caveat: Packer core only links a parent version when the template also resolves
that same external identifier through an `hcp_packer_artifact` data source; the
link is matched on that exact string. A user must wire up the data source for the
ancestry to appear.

Why it is a separate change: it does nothing for ISO-install builds, which have
no parent version to link. It is parity work for completeness, not a bug fix.