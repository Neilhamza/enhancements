---
title: microshift-kubelet-image-credential-provider
authors:
  - "@Neilhamza"
reviewers:
  - "@pacevedom"
  - "@eslutsky"
  - "@copejon"
  - "@ggiguash"
  - "@pmtk"
  - "@kasturinarra"
  - "@qJkee"
  - "@dfroehli"
approvers:
  - "@jerpeter1"
api-approvers:
  - None
creation-date: 2026-09-02
last-updated: 2026-09-02
tracking-link:
  - https://redhat.atlassian.net/browse/OCPSTRAT-3456
see-also:
  - enhancements/microshift/enabling-user-specified-featuregates.md
  - enhancements/microshift/microshift-dns-resource-configuration.md
---

# MicroShift Kubelet Image Credential Provider Configuration

## Summary

MicroShift runs kubelet as an embedded, in-process component. Kubelet's [image credential provider](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/) feature (GA since Kubernetes 1.26) allows kubelet to obtain container registry credentials dynamically at image pull time by executing an external provider binary. The feature is enabled through two kubelet command-line flags, `--image-credential-provider-config` and `--image-credential-provider-bin-dir`, which have no equivalent in `KubeletConfiguration`. Because MicroShift exposes no kubelet command line and only maps its `kubelet:` configuration section into `KubeletConfiguration`, users currently have no way to enable this feature.

This enhancement introduces two new optional keys, `imageCredentialProviderConfigPath` and `imageCredentialProviderBinDir`, under the existing `kubelet:` section of the MicroShift configuration file. MicroShift extracts these keys, validates them at startup, and applies them to the embedded kubelet's startup flags.

## Motivation

Edge devices frequently pull application images from private, token-based registries such as Amazon ECR, Google Artifact Registry, or Azure Container Registry. These registries do not issue long-lived passwords; Amazon ECR, for example, issues authorization tokens that expire after 12 hours.

Today, MicroShift users work around this with a systemd timer that periodically obtains a fresh token (e.g. `aws ecr get-login-password`) and rewrites it into CRI-O's static auth file. This workaround has known drawbacks:

- **Token gap:** if the timer fires late, or the token expires early, image pulls fail until the next refresh.
- **Operational overhead:** an additional systemd unit and script must be maintained on every device.
- **Not portable:** each registry type (ECR, GCR, ACR) requires its own bespoke script.

The kubelet image credential provider is the upstream-standard, cloud-vendor-recommended solution to this problem. A customer (see the linked RFE) requires it to pull private images from Amazon ECR on MicroShift edge devices.

### User Stories

1. As a MicroShift administrator, I want to enable the kubelet image credential provider through the MicroShift configuration file so that my edge devices can authenticate to private registries at pull time without direct access to kubelet command-line flags.
2. As a MicroShift administrator with devices pulling from Amazon ECR, I want kubelet to obtain short-lived registry credentials automatically so that I no longer maintain a per-device token rotation script.
3. As a MicroShift administrator, I want MicroShift to fail to start with a clear error if the credential provider configuration points to paths that do not exist, so that misconfigurations are caught at startup rather than surfacing as opaque image pull failures.

### Goals

1. Allow users to set `imageCredentialProviderConfigPath` and `imageCredentialProviderBinDir` under the `kubelet:` section of the MicroShift configuration file.
2. Apply these settings to the embedded kubelet as startup flags when present.
3. Preserve current behavior when the keys are not set, ensuring full backward compatibility.
4. Validate the provided paths at startup and fail with a clear error message if they are invalid.

### Non-Goals

1. A generic mechanism for passing arbitrary kubelet command-line flags through the MicroShift configuration file. This enhancement special-cases exactly two keys.
2. Packaging or distributing credential provider binaries (e.g. `ecr-credential-provider`). These are standalone upstream binaries that users install separately.
3. Validating the contents of the credential provider configuration file or the presence and executability of binaries in the bin directory. Kubelet performs this validation itself.
4. Runtime reconfiguration without restarting MicroShift.

## Proposal

Introduce two new optional keys under the existing `kubelet:` section of the MicroShift configuration file:

```yaml
kubelet:
  imageCredentialProviderConfigPath: /etc/microshift/credential-providers.yaml
  imageCredentialProviderBinDir: /usr/libexec/microshift/credential-providers
```

The `kubelet:` section is currently a schemaless map (`map[string]any`) whose contents are marshaled verbatim into the generated `KubeletConfiguration` file. The two new keys are kubelet **flags**, not `KubeletConfiguration` fields, so they cannot follow that path: if left in the map, kubelet's strict decoder rejects them, falls back to a lenient decoder, logs a warning, and silently ignores them. MicroShift will therefore read these two keys from the map during configuration processing into typed internal fields, validate them, and set them on the `KubeletFlags` struct used to start the embedded kubelet. When the remaining `kubelet:` contents are marshaled into `KubeletConfiguration`, the two reserved keys are filtered out. The `Config.Kubelet` map itself is not modified, so `microshift show-config` continues to display the keys exactly where the user set them. All other keys in the `kubelet:` section continue to be passed through to `KubeletConfiguration` unchanged.

### Workflow Description

1. MicroShift starts up and loads its configuration file. The `kubelet:` section is read into `Config.Kubelet` as a schemaless map.
2. During computed-value processing, MicroShift reads `imageCredentialProviderConfigPath` and `imageCredentialProviderBinDir` from the map into typed internal fields. The map is not modified. An empty string is treated as unset.
3. During validation, MicroShift checks that either both keys are set or neither is, that both paths are absolute, that `imageCredentialProviderConfigPath` exists as a file or directory, and that `imageCredentialProviderBinDir` exists and is a directory.
4. If validation fails, MicroShift exits with a descriptive error message naming the field and the path.
5. If validation succeeds, the kubelet component sets `KubeletFlags.ImageCredentialProviderConfigPath` and `KubeletFlags.ImageCredentialProviderBinDir` before starting the embedded kubelet, and logs an informational message recording the configured values.
6. The keys in the `kubelet:` section other than the two reserved keys are marshaled into the `KubeletConfiguration` file as today.
7. At image pull time, kubelet consults the credential provider configuration; for images matching a configured provider, kubelet executes the provider binary, caches the returned credentials in memory for the returned or default cache duration, and passes them to CRI-O with the pull request.
8. If the keys are not set, kubelet starts without credential provider configuration, identical to current behavior.

### API Extensions

The following changes to the MicroShift configuration file are proposed:

```yaml
kubelet:
  imageCredentialProviderConfigPath: <string>  # optional, default: not set
  imageCredentialProviderBinDir: <string>      # optional, default: not set
```

- `imageCredentialProviderConfigPath`: path to a kubelet `CredentialProviderConfig` file (JSON or YAML), or a directory of such files which kubelet merges in lexicographical order.
- `imageCredentialProviderBinDir`: path to the directory containing credential provider plugin binaries.

Both keys must be set together; setting only one is a validation error. Values must be absolute paths. An empty string is equivalent to omitting the key. Because user configuration files and drop-ins under `/etc/microshift/config.d/` are combined with a JSON merge patch, which merges maps key by key, the two keys may be set in different files (for example, one in `config.yaml` and the other in a drop-in) and are merged before validation.

The `kubelet:` section is annotated `+kubebuilder:validation:Schemaless`, so no change to the generated configuration schema or a schema version bump is required. As a consequence, the configuration generator cannot emit the keys as schema entries; they are documented through the doc comment on the `Kubelet` field, which the generator propagates into the sample configuration file and the configuration reference (see Sample configuration and documentation below).

`microshift show-config --mode effective` displays the keys under `kubelet:` as set by the user, since the underlying map is not modified.

The credential provider configuration file itself follows the upstream `kubelet.config.k8s.io/v1` `CredentialProviderConfig` format. For example, for Amazon ECR:

```yaml
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
providers:
  - name: ecr-credential-provider
    matchImages:
      - "*.dkr.ecr.*.amazonaws.com"
    defaultCacheDuration: "6h"
    apiVersion: credentialprovider.kubelet.k8s.io/v1
```

### Topology Considerations

#### Hypershift / Hosted Control Planes
N/A

#### Standalone Clusters
N/A

#### Single-node Deployments or MicroShift
Enhancement is intended for MicroShift only.

#### OpenShift Kubernetes Engine
N/A

### Implementation Details/Notes/Constraints

#### Config struct changes

Add two internal, non-serialized fields to the `Config` struct in `pkg/config/config.go`. These follow the existing convention for internal fields (e.g. `userSettings`) and are populated by reading the schemaless map rather than by direct decoding. The `Kubelet` field's doc comment is extended because the configuration generator propagates it into the sample configuration and reference documentation:

```go
type Config struct {
    // ... existing fields ...

    // Settings specified in this section are transferred as-is into the Kubelet config,
    // with two exceptions that are applied as kubelet startup flags instead:
    //   imageCredentialProviderConfigPath: absolute path to a kubelet CredentialProviderConfig
    //     file, or a directory of such files. Enables the kubelet image credential provider.
    //   imageCredentialProviderBinDir: absolute path to the directory containing credential
    //     provider plugin binaries. Must be set together with imageCredentialProviderConfigPath.
    // +kubebuilder:validation:Schemaless
    Kubelet map[string]any `json:"kubelet"`

    // Read from the Kubelet map during updateComputedValues().
    // These are kubelet flags, not KubeletConfiguration fields.
    KubeletImageCredentialProviderConfigPath string `json:"-"`
    KubeletImageCredentialProviderBinDir     string `json:"-"`
}
```

Populating internal fields from the schemaless `Kubelet` map is a new pattern in the MicroShift configuration code. The map has always been passed through untouched, and this enhancement preserves that: the map is read, not modified.

#### Config extraction

Extend `updateComputedValues()` in `pkg/config/config.go` to read the two keys. The map is not modified, so `incorporateUserSettings()` (which assigns `c.Kubelet = u.Kubelet`) and `microshift show-config` are unaffected:

```go
const (
    kubeletImageCredentialProviderConfigPathKey = "imageCredentialProviderConfigPath"
    kubeletImageCredentialProviderBinDirKey     = "imageCredentialProviderBinDir"
)

// kubeletReservedKeys are keys under `kubelet:` that are applied as kubelet
// startup flags rather than passed through to KubeletConfiguration.
var kubeletReservedKeys = []string{
    kubeletImageCredentialProviderConfigPathKey,
    kubeletImageCredentialProviderBinDirKey,
}

func (c *Config) readKubeletCredentialProviderSettings() error {
    for key, dst := range map[string]*string{
        kubeletImageCredentialProviderConfigPathKey: &c.KubeletImageCredentialProviderConfigPath,
        kubeletImageCredentialProviderBinDirKey:     &c.KubeletImageCredentialProviderBinDir,
    } {
        raw, ok := c.Kubelet[key]
        if !ok || raw == nil {
            continue
        }
        s, ok := raw.(string)
        if !ok {
            return fmt.Errorf("kubelet.%s must be a string, got %T", key, raw)
        }
        *dst = s
    }
    return nil
}

// KubeletPassthrough returns the user-provided kubelet settings that are
// transferred into KubeletConfiguration, excluding reserved keys.
func (c *Config) KubeletPassthrough() map[string]any {
    if c.Kubelet == nil {
        return nil
    }
    out := make(map[string]any, len(c.Kubelet))
    for k, v := range c.Kubelet {
        out[k] = v
    }
    for _, k := range kubeletReservedKeys {
        delete(out, k)
    }
    return out
}
```

`updateComputedValues()` calls `readKubeletCredentialProviderSettings()` before returning. `generateConfig()` in `pkg/node/kubelet.go` is changed to marshal `cfg.KubeletPassthrough()` instead of `cfg.Kubelet`, so the reserved keys never reach the `KubeletConfiguration` file. The list of reserved keys lives only in the `config` package.

#### Config validation

Extend `validate()` in `pkg/config/config.go`:

```go
func (c *Config) validateKubeletCredentialProvider() error {
    configPath := c.KubeletImageCredentialProviderConfigPath
    binDir := c.KubeletImageCredentialProviderBinDir

    if configPath == "" && binDir == "" {
        return nil
    }
    if configPath == "" || binDir == "" {
        return fmt.Errorf("kubelet.%s and kubelet.%s must be set together",
            kubeletImageCredentialProviderConfigPathKey, kubeletImageCredentialProviderBinDirKey)
    }
    if !filepath.IsAbs(configPath) {
        return fmt.Errorf("kubelet.%s (%q) must be an absolute path",
            kubeletImageCredentialProviderConfigPathKey, configPath)
    }
    if !filepath.IsAbs(binDir) {
        return fmt.Errorf("kubelet.%s (%q) must be an absolute path",
            kubeletImageCredentialProviderBinDirKey, binDir)
    }

    if _, err := os.Stat(configPath); err != nil {
        return fmt.Errorf("error validating kubelet.%s (%q): file or directory does not exist: %w",
            kubeletImageCredentialProviderConfigPathKey, configPath, err)
    }

    info, err := os.Stat(binDir)
    if err != nil {
        return fmt.Errorf("error validating kubelet.%s (%q): directory does not exist: %w",
            kubeletImageCredentialProviderBinDirKey, binDir, err)
    }
    if !info.IsDir() {
        return fmt.Errorf("error validating kubelet.%s (%q): not a directory",
            kubeletImageCredentialProviderBinDirKey, binDir)
    }

    return nil
}
```

Validation rules:

- Both-or-neither: setting only one of the two keys is an error. Kubelet requires both to activate the feature; failing early in MicroShift produces a clearer message than kubelet's own error. An empty string is treated as unset.
- Both paths must be absolute. A relative path would be resolved against the MicroShift process working directory, which is not a stable location.
- `imageCredentialProviderConfigPath` must exist. A file or a directory is accepted, matching upstream kubelet semantics.
- `imageCredentialProviderBinDir` must exist and be a directory.
- Error messages name the field and the path.

MicroShift deliberately does **not** validate the contents of the credential provider configuration file, nor the presence or executability of binaries inside the bin directory. Kubelet validates these at startup and its errors surface in the journal. Duplicating that validation would require keeping MicroShift in sync with upstream kubelet behavior.

Optionally, MicroShift may emit a warning via `AddWarning()` if the bin directory is writable by users other than root, since credential provider binaries execute with kubelet's privileges. This is a warning, not a startup failure.

#### Kubelet flag injection

Extend `configure()` in `pkg/node/kubelet.go`. The `KubeletFlags` struct from the vendored `k8s.io/kubernetes/cmd/kubelet/app/options` package already exposes the required fields, and they are passed through to `NewMainKubelet()` by the existing in-process startup path. No new plumbing is required:

```go
func (s *KubeletServer) configure(cfg *config.Config) {
    // ... existing code ...
    kubeletFlags := kubeletoptions.NewKubeletFlags()
    // ... existing flag assignments ...

    if cfg.KubeletImageCredentialProviderConfigPath != "" {
        kubeletFlags.ImageCredentialProviderConfigPath = cfg.KubeletImageCredentialProviderConfigPath
        kubeletFlags.ImageCredentialProviderBinDir = cfg.KubeletImageCredentialProviderBinDir
        klog.InfoS("Kubelet image credential provider enabled",
            "configPath", cfg.KubeletImageCredentialProviderConfigPath,
            "binDir", cfg.KubeletImageCredentialProviderBinDir)
    }

    // ... existing code ...
}
```

The explicit log line is the intended verification point for tests and support. Because MicroShift calls `kubelet.Run()` directly rather than through kubelet's cobra command, kubelet's usual `FLAG: --image-credential-provider-config=...` startup lines are never emitted, and kubelet's own credential provider code logs nothing at default verbosity when plugins are registered.

#### Sample configuration and documentation

`packaging/microshift/config.yaml` and `docs/user/howto_config.md` are generated by `scripts/generate-config.sh` and kept in sync by `scripts/verify/verify-config.sh`; they must not be edited by hand. Because the `kubelet:` section is schemaless, the generator cannot emit the two keys as schema entries. They are documented by extending the doc comment on the `Kubelet` field in `pkg/config/config.go` (shown above), which the generator propagates into both files. After changing the comment, run `make generate-config` and commit the regenerated files.

### Risks and Mitigations

**Risk:** A user places the keys under `kubelet:` in a MicroShift version that supports the feature, but a typo in the key name (e.g. `imageCredentialProviderConfig`) causes the key to fall through to the `KubeletConfiguration` passthrough, where kubelet's lenient decoder silently ignores it. The feature appears enabled but is inert.
**Mitigation:** Kubelet logs a lenient-decode warning in the journal. A future improvement could warn on near-miss keys. This risk is not specific to this enhancement; it applies to any misspelled key in the passthrough.

**Risk:** Kubelet's lenient decoding of unknown `KubeletConfiguration` fields is marked upstream as temporary, to be removed when `v1beta1` support is dropped. When that happens, any unknown key left in the passthrough becomes a hard kubelet startup failure.
**Mitigation:** This enhancement extracts its two keys in the MicroShift configuration layer before the map is marshaled, so it never depends on lenient decoding. The general passthrough exposure predates this enhancement.

**Risk:** Credential provider binaries execute with kubelet's (root) privileges. A bin directory writable by unprivileged users is a local privilege escalation vector.
**Mitigation:** User documentation must state this trust boundary and recommend a root-owned bin directory that is not writable by other users. MicroShift may additionally emit a startup warning for world-writable bin directories.

**Risk:** On image-based (ostree/bootc) systems, a validation failure at startup causes greenboot health checks to fail, triggering an automatic rollback to the previous deployment.
**Mitigation:** This is the intended composition of fail-fast validation with greenboot: the device returns to a known-good state rather than remaining unhealthy. Documentation must note that the provider config file and bin directory must exist in the image before the configuration keys are enabled.

**Risk:** Users may set `defaultCacheDuration` in the provider configuration longer than the registry token lifetime, causing kubelet to serve expired credentials.
**Mitigation:** Documentation for Amazon ECR (12-hour tokens) must recommend a cache duration comfortably shorter than the token lifetime.

### Drawbacks

Special-casing two keys inside an otherwise schemaless passthrough section introduces a small amount of hidden behavior: two keys under `kubelet:` are treated differently from all others, and the generated schema cannot express them. This is accepted because the OCPSTRAT acceptance criteria specify placement under `kubelet:`, and the alternative of a generic flag passthrough was explicitly ruled out of scope.

## Test Plan

### Unit Tests

- Extraction: both keys present, both absent, only one present.
- Extraction: non-string values rejected with an error.
- Extraction: `c.Kubelet` is not modified; `KubeletPassthrough()` returns the map without the two reserved keys and with all other keys intact.
- Extraction: empty string values are treated as unset.
- Validation: only one key set fails with the both-or-neither error.
- Validation: relative paths are rejected.
- Validation: `imageCredentialProviderConfigPath` missing fails; existing file passes; existing directory passes.
- Validation: `imageCredentialProviderBinDir` missing fails; path is a file fails; existing directory passes.
- Validation: neither key set passes (backward compatibility).
- `generateConfig()`: the two keys never appear in the generated `KubeletConfiguration` YAML; other user-provided keys still do.
- `configure()`: `KubeletFlags` carries both values when set, and empty values when unset.

### Integration Tests (Robot Framework)

- Default behavior: no keys set, MicroShift starts, the "Kubelet image credential provider enabled" log line is absent.
- Valid configuration: both keys set to an existing file and directory via drop-in, restart, verify MicroShift starts and the journal contains the "Kubelet image credential provider enabled" log line with the configured paths.
- `show-config`: verify `microshift show-config --mode effective` displays both keys under `kubelet:`.
- Split configuration: one key in `config.yaml` and the other in a `/etc/microshift/config.d/` drop-in, verify the merged configuration is valid and MicroShift starts.
- Empty strings: both keys set to `""`, verify behavior is identical to omitting them.
- Relative path: verify MicroShift fails to start with an error naming the field.
- Directory config path: `imageCredentialProviderConfigPath` set to a directory, verify MicroShift starts.
- Missing config path: verify MicroShift fails to start with an error naming the field and path in the journal.
- Missing bin directory: verify MicroShift fails to start with an error naming the field and path in the journal.
- Only one key set: verify MicroShift fails to start with the both-or-neither error.
- Recovery: restore valid configuration, restart, verify MicroShift starts.
- Downgrade: configuration with the new keys on a MicroShift version without the feature, verify MicroShift starts with a kubelet lenient-decode warning and the feature inert.
- End-to-end pull: deploy a mock credential provider (an executable implementing the `credentialprovider.kubelet.k8s.io/v1` exec contract returning static credentials) and a local password-protected registry, deploy a pod referencing a private image with no CRI-O auth pre-populated, verify the pod reaches Running.
- End-to-end negative: mock provider returns incorrect credentials, verify the pod reaches ImagePullBackOff.
- Coexistence with CRI-O auth: with the credential provider configured and CRI-O auth also pre-populated for the same registry, verify the pull succeeds.
- Mixed registries: pods referencing an image from a credential-provider registry and an image from a registry using CRI-O static auth, verify both pulls succeed independently.
- Caching: with a short `defaultCacheDuration`, verify a second pull within the cache window does not re-invoke the provider and a pull after expiry does.

A one-time manual validation against Amazon ECR with the upstream `ecr-credential-provider` binary is performed before release to confirm the customer's exact path and to source the configuration guide. Automated CI coverage uses the mock provider and local registry only.

## Graduation Criteria

### Dev Preview -> Tech Preview
N/A

### Tech Preview -> GA

- Ability to utilize the enhancement end to end
- End user documentation completed and published, including a configuration guide for Amazon ECR
- Available by default
- End-to-end tests
- Unit tests covering config extraction and validation

### Removing a deprecated feature
N/A

## Upgrade / Downgrade Strategy

When upgrading from a version without support for these keys, the keys remain unset unless the user adds them, and existing behavior is preserved. If a user pre-stages the keys in the configuration file before upgrading, the older version ignores them and the newer version activates them on its first start. No migration is required; the feature holds no persisted state.

When downgrading to a version without support for these keys, the older version passes them through to the `KubeletConfiguration` file, where kubelet's lenient decoder logs a warning and ignores them. MicroShift starts normally with the credential provider feature inactive. Private registry image pulls will fail on the downgraded version until the previous workaround is restored or the device is upgraded again. The configuration file on disk is not modified. This behavior depends on kubelet retaining lenient decoding for `v1beta1`, which holds for all currently supported downgrade targets.

## Version Skew Strategy
N/A

## Operational Aspects of API Extensions

The credential provider is invoked by kubelet only for images matching a configured `matchImages` pattern; pulls from other registries are unaffected. Credentials are held in kubelet's memory only, never written to disk, and are discarded when MicroShift restarts; the first matching pull after a restart re-invokes the provider.

The provider binary must be able to authenticate to the cloud provider itself (for Amazon ECR: an EC2 instance role, IAM Roles Anywhere, or static credentials in `/root/.aws/credentials`). Failures in the provider surface as image pull failures (`ImagePullBackOff`) with the provider's error in the kubelet journal.

Changes to the two configuration keys require a MicroShift restart. Changes to the provider configuration file or binaries require a MicroShift restart for kubelet to reload them.

## Support Procedures

- Confirm the keys were applied: `journalctl -u microshift` at startup contains `Kubelet image credential provider enabled` with the configured `configPath` and `binDir`. Kubelet does not log `FLAG:` lines in MicroShift and logs nothing at default verbosity when providers are registered, so this MicroShift log line is the authoritative signal.
- Confirm the effective configuration: `microshift show-config --mode effective` displays both keys under `kubelet:`.
- Startup validation failures are logged with the field name and path, e.g. `error validating kubelet.imageCredentialProviderConfigPath ("/etc/microshift/credential-providers.yaml"): file or directory does not exist`.
- Image pull failures with the feature active: inspect `journalctl -u microshift` for provider execution errors, verify the provider binary runs successfully when invoked manually with a `CredentialProviderRequest` on stdin, and verify the device's cloud identity has registry pull permissions.
- A lenient-decode warning in the kubelet logs referencing either key indicates a MicroShift version that does not support the feature, or a misspelled key.

## Alternatives (Not Implemented)

**Generic kubelet flag passthrough.** A `kubelet.args` or similar mechanism for arbitrary flags was rejected. It exposes the full kubelet flag surface, most of which MicroShift manages deliberately, and increases the risk of misconfiguration. The OCPSTRAT explicitly rules this out of scope; it should be tracked as a separate RFE if needed.

**Typed `kubelet:` section.** Replacing the schemaless map with a typed struct was rejected because it would break the existing passthrough of arbitrary `KubeletConfiguration` fields, which users depend on.

**Separate top-level configuration section.** Placing the keys outside `kubelet:` (e.g. `imageCredentialProvider:`) would avoid special-casing keys inside the passthrough and would be picked up by schema generation. It was rejected because the OCPSTRAT acceptance criteria and the originating RFE specify placement under `kubelet:`, where users familiar with the upstream flags will look for them.

**Removing the reserved keys from the `Config.Kubelet` map.** An earlier draft extracted the two keys by deleting them from (a copy of) the map during configuration processing. This was rejected because `microshift show-config --mode effective` marshals the `Config` struct, so the keys would disappear from its output. Filtering at the point where the map is marshaled into `KubeletConfiguration` keeps the effective configuration faithful to user input.

**Packaging credential provider binaries.** Shipping `ecr-credential-provider` or equivalents as MicroShift RPMs was rejected. The binaries are cloud-specific, upstream-maintained, and have their own release and CVE cadence. Bundling them would make their lifecycle a MicroShift concern.
