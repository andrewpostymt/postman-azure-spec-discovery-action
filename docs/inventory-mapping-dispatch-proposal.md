# Inventory, Mapping, And Dispatch Proposal

Status: Draft
Owner: Postman CSE
Review outcome: Pending

## Executive Decision

Extend `postman-azure-spec-discovery-action` from a repository-local Azure
spec resolver into the shared inventory engine for estate-scale onboarding, but
do not make the Azure action own every dispatch concern directly.

The selected direction is:

- For active application repositories, make build-generated OpenAPI artifacts
  the primary source of truth. For .NET services, this means CI produces a
  deterministic `swagger.json` from the `.csproj` build path before promotion.
- Keep this repository as the authoritative discovery and normalization layer
  for Azure-hosted API surfaces that need inventory, mapping, backfill, or
  gateway-state correlation.
- Add provider-neutral inventory and mapping schemas that can represent APIM,
  API Center, App Service, WSO2, Apigee, Kong, MuleSoft, AWS API Gateway, and
  future gateway sources without changing downstream pipeline contracts.
- Move ADO-specific queueing into a dispatch adapter or companion package, not
  into provider implementations.
- Treat approved mapping state as the boundary between discovery suggestions
  and automation.
- Keep broad estate discovery in a controller repository. Target service repos
  should receive exact source identity or a spec artifact and should not scan
  the whole estate.
- Treat gateway contracts, including WSO2 gateway artifacts, as downstream
  deployment projections unless the gateway is the only available contract
  source for a legacy service.

This keeps the proven `postman-ado-spec-discovery` controller model while
removing its APIM-specific assumptions.

## Existing-System Inventory

| Asset | Authoritative location | Current role | Reuse/change decision |
| --- | --- | --- | --- |
| Azure discovery action | `postman-cs/postman-azure-spec-discovery-action` | Resolves one repo-local spec, exports many candidates, or enumerates repo associations from Azure Resource Graph | Extend with normalized inventory/mapping schemas and provider-neutral source identity |
| Provider registry | `src/lib/providers/registry.ts` | Registers Azure provider identity, source types, formats, probe order, and capability text | Reuse as the Azure provider registry; do not make it the global gateway registry |
| Provider interface | `src/lib/providers/types.ts` | `SpecProvider` with probe, list, hydrate, export seams | Extend or wrap with a provider-neutral inventory adapter seam |
| Estate enumerator | `src/lib/estate/enumerate.ts` | Association-only repo roster from Azure tags | Reuse as one estate evidence source, but it is not enough for mapping or dispatch |
| ADO spec discovery | `postman-ado-spec-discovery` | APIM-oriented ADO inventory, approved mapping, and dispatch proof point | Use as the lifecycle model to generalize, not as the provider model |
| ADO pipeline templates | `postman-ado-pipeline-templates` | Target repo onboarding/bootstrap/repo-sync execution | Add support for one structured `specSourceJson` or exported spec artifact input |

Known-good evidence:

- `postman-azure-spec-discovery-action` already supports `resolve-one`,
  `discover-many`, and association-only `discover-estate`.
- `postman-ado-spec-discovery` has demonstrated the controller repository
  shape: inventory artifact, approval mapping, and dispatch of existing target
  onboarding pipelines.
- Representative ADO onboarding validation showed that target pipelines should
  avoid re-discovering broad state when an exact spec/source identity is known.

## Problem Evidence

Observed behavior:

- `postman-azure-spec-discovery-action` can discover Azure candidates but does
  not own the full inventory -> mapping -> approved dispatch lifecycle.
- `postman-ado-spec-discovery` owns that lifecycle but is modeled around Azure
  APIM names, parameters, and schemas.
- Customer estates may include APIM, WSO2, API Center, runtime-declared specs,
  or services with no authored spec. A hardcoded APIM pipeline does not transfer
  cleanly across those environments.
- Active service teams can produce a more deterministic OpenAPI contract during
  build than by reverse-exporting from a gateway after deployment.

Expected behavior:

- One controller workflow inventories all visible API/spec sources from the
  available control planes.
- Active repositories generate and validate `swagger.json` before promotion to
  development environments.
- The workflow emits normalized inventory and mapping files.
- Humans approve or correct mappings where metadata confidence is not exact.
- Approved mappings dispatch downstream onboarding pipelines with exact source
  identity or a previously exported spec artifact.
- Target repositories consume a generic source contract instead of APIM-specific
  parameters.

Impact:

- Without this split, each target repo needs bespoke gateway discovery logic.
- Broad discovery per target repo slows rollout and increases ambiguity.
- Weak metadata can accidentally connect the wrong API to the wrong repository
  unless mapping approval is explicit.

## Scope

In scope:

- Normalized inventory schema.
- Normalized mapping schema with explicit approval.
- Provider-neutral source identity object.
- ADO dispatch adapter contract.
- Evidence and confidence model for repository ownership.
- Migration path from APIM-specific ADO discovery schemas.
- Documentation of unsupported and partial-spec cases.
- Build-generated OpenAPI as the preferred source for active services.
- Gateway-discovery backfill for legacy or no-new-work services.

Out of scope:

- Inferring complete OpenAPI contracts from arbitrary live traffic.
- Provisioning new target repositories.
- Creating Postman workspaces directly from the controller repo.
- Mutating gateway resources.
- Auto-approving weak name/path matches.
- Replacing existing target onboarding templates in the first iteration.
- Treating WSO2 export as deterministic when the same service can generate
  OpenAPI from source during build.

Assumptions and unknowns:

- Metadata quality varies by customer and gateway.
- Gateway tags may be missing, stale, duplicated, or applied at the wrong level.
- Some gateways may expose inventory but not exportable specifications.
- Some services may have no spec in the gateway at all.
- Some active services may need `.csproj` or build-pipeline updates before they
  can emit `swagger.json` reliably.
- ADO repository ownership may not be available from gateway metadata and may
  require a manual mapping step or an external service catalog.

## Invariants

| ID | Invariant | Enforcement | Verification |
| --- | --- | --- | --- |
| INV-1 | Discovery never mutates gateway resources | Provider adapters expose read/export only; no create/update/delete commands | Unit tests reject mutation calls; live validation confirms read-only roles |
| INV-2 | Names are never durable identity by themselves | Inventory stores provider type, gateway id, API id, version/revision, and source provenance | Mapping tests cover rename and duplicate-name cases |
| INV-3 | Weak evidence never auto-dispatches | Dispatch filters require `approved: true`; weak confidence starts unapproved | Mapping merge tests and dispatch tests |
| INV-4 | Broad estate scans run in controller workflows, not target repos | Target contract accepts exact source identity or artifact path | Pipeline contract tests |
| INV-5 | Re-running inventory is idempotent | Stable keys, content hashes, schema versions, and merge-preserved approvals | Golden inventory/mapping tests |
| INV-6 | Concurrent dispatches do not write the same target branch at the same time | Dispatcher serializes by target repo + branch | Dispatch concurrency tests |
| INV-7 | Secrets and private URLs are not serialized into inventory or logs | Sanitizers and schema-level redaction rules | Sanitization tests and fixture review |
| INV-8 | Partial or reconstructed specs are labeled as such | `contractClass` and completeness fields survive inventory and dispatch | Contract tests across provider outputs |
| INV-9 | Deleted or missing APIs are retired, not silently removed | Mapping merge marks removed entries unapproved | Migration and refresh tests |
| INV-10 | Existing v1 action behavior remains compatible | New modes and outputs are additive by default | Action contract and README table tests |

## What Changes Compared To `postman-ado-spec-discovery`

`postman-ado-spec-discovery` is the right operating model but the wrong
abstraction boundary for full-scale gateway portability.

Keep from `postman-ado-spec-discovery`:

- Controller repository as the estate inventory owner.
- Generated inventory artifacts.
- Approval mapping file.
- Dispatch approved mappings only.
- Serial dispatch per target repository and branch.
- Target pipeline receives exact gateway API identity or spec path.

Change in `postman-azure-spec-discovery-action`:

- Replace APIM-specific inventory schemas with provider-neutral schemas.
- Replace `apimResourceGroup`, `apimServiceName`, and `apimApiId` as the primary
  public contract with `specSourceJson`.
- Treat APIM as one provider adapter, not the model every gateway must copy.
- Add mapping confidence and evidence classes as first-class schema fields.
- Add `inventory` and `mapping` artifacts that can contain multiple providers.
- Add a dispatch adapter interface so ADO, GitHub Actions, or another runner can
  queue target automation from the same approved mapping.
- Keep Azure-specific Resource Graph logic inside the Azure adapter layer.

The Azure repo should not blindly absorb all of ADO discovery. The cleaner path
is either:

- extract a shared package such as `@postman-cse/spec-discovery-core`, then make
  both repos use it; or
- make this repository the first implementation of the core contracts while
  keeping ADO dispatch as an adapter module.

The first option is cleaner if WSO2 and non-Azure gateways are real near-term
targets. The second option is faster but risks making an Azure-named repository
carry non-Azure semantics.

## Packaging And Ownership Model

If `postman-azure-spec-discovery-action` is tuned up and
`postman-ado-spec-discovery` becomes the ADO transport layer, the split should
be:

- `postman-azure-spec-discovery-action` owns discoverability for Azure-hosted
  surfaces: Azure auth, provider probing, candidate enumeration, export,
  contract fidelity, evidence, and normalized inventory output.
- A shared core owns provider-neutral schemas and pure lifecycle logic:
  inventory normalization, mapping merge, confidence classification, removed
  API handling, and source-object generation.
- `postman-ado-spec-discovery` owns ADO orchestration: controller pipelines,
  artifact publication, optional commit of inventory/mapping files, ADO pipeline
  queueing, ADO run correlation, and target repo/branch serialization.
- Target onboarding templates own Postman asset creation, repo-sync, lint,
  collection runs, mocks, monitors, and target repository writes.

In that model, `postman-ado-spec-discovery` is a wrapper/controller around the
discoverability layer. It should not be embedded inside Azure provider code. It
may consume the discovery layer in one of two ways:

1. CLI composition:
   - ADO pipeline authenticates with `AzureCLI@2` or workload identity.
   - ADO controller calls the Azure discovery CLI in inventory mode.
   - The CLI writes normalized `inventory.json`, exported specs, and suggested
     `mapping.json`.
   - ADO controller merges/commits/publishes those artifacts and dispatches
     approved targets.

2. Package composition:
   - A shared `@postman-cse/spec-discovery-core` package exposes schemas,
     mappers, merge logic, and provider interfaces.
   - The Azure discovery action imports the core and registers Azure providers.
   - The ADO controller imports the core and ADO dispatch adapter.
   - Future gateway adapters, such as WSO2, can register against the same core
     without becoming Azure-specific code.

The CLI composition path is the lower-friction first step because it does not
force an immediate monorepo/package split. The package composition path is the
better long-term shape if non-Azure gateways are expected to be first-class.

Under either path, ADO remains transport/controller, not discovery truth. It
should not run `az apim` directly once Azure discovery can emit the same
normalized inventory. Its job becomes:

- select the controller scope;
- call the discoverability layer;
- preserve approved mapping state;
- publish reviewable artifacts;
- queue approved downstream pipelines;
- record dispatch results.

## Operating Model Shift

The operating model changes from "each repository figures out its own API" to
"a controller repository inventories the estate, then dispatches exact approved
work."

Current single-repo model:

- Target repo runs discovery.
- Target repo resolves one spec.
- Target repo creates or updates Postman assets.
- Target repo owns `.postman` state.

Proposed estate model:

- Controller repo runs discovery across a selected gateway or subscription
  scope.
- Controller repo writes inventory and suggested mappings.
- Human or platform owner approves mappings.
- Controller dispatches target pipelines only for approved mappings.
- Target repo receives `specSourceJson` or an exported spec artifact and runs
  normal onboarding without broad estate discovery.

This is the important scaling behavior: broad discovery happens once per
controller run, while each target repo receives exact, bounded work.

## Build-Generated Contract Policy

For active application repositories, the preferred policy is not WSO2 -> OAS
discovery. It is source/build -> OAS -> downstream consumers.

For .NET services, that means:

- the application repository owns `.csproj` changes required to emit
  `swagger.json`;
- CI generates `swagger.json` as part of build or pre-deploy validation;
- the onboarding pipeline discovers generated `swagger.json`, `swagger.yaml`,
  or `swagger.yml` artifacts instead of requiring each repository to hardcode a
  spec path;
- the pipeline validates the generated document before promotion to `dev`;
- Postman onboarding consumes that generated OpenAPI artifact;
- WSO2 deployment consumes a translated gateway contract derived from the same
  validated artifact.

This makes the generated OpenAPI document the shared contract source for active
development. WSO2 is then a deployment consumer and policy-enforcement surface,
not the authoritative contract source.

WSO2 -> OAS export remains useful only for bounded cases:

- legacy services with no planned code changes;
- initial inventory/backfill where no repository-generated spec exists yet;
- drift detection between gateway state and source-generated contract;
- migration assessment before the service adopts build-generated OpenAPI.

It should not be described as deterministic unless the gateway stores and
exports a complete authored specification with stable semantics. Gateway exports
can be stale, transformed, policy-shaped, partial, or missing implementation
detail. Those are valid discovery signals, but weaker contract authority than a
spec generated from the application build.

The rollout policy becomes:

- active services must emit valid OpenAPI in CI before push or promotion to
  `dev`;
- standard onboarding templates should resolve the generated spec path by
  scanning known build artifact locations before Postman or WSO2 steps run;
- nightly builds should regenerate and validate specs to measure coverage and
  catch drift;
- services without active development use the controller/backfill pipeline to
  inventory, extract, or bootstrap specs;
- both Postman and WSO2 consume the same validated artifact whenever possible.

Spec discovery inside the target pipeline should be deterministic:

- include only generated OpenAPI candidates matching approved names such as
  `swagger.json`, `swagger.yaml`, and `swagger.yml`;
- scan only bounded repository/build-output locations, not the whole working
  tree by default;
- ignore dependency, package, build-cache, and generated client folders;
- select automatically only when exactly one valid candidate remains;
- allow a committed binding or explicit selector only for repositories with
  multiple intentional specs;
- fail before gateway/Postman mutation when no candidate or multiple ambiguous
  candidates remain;
- emit the resolved spec path as the pipeline output consumed by Postman
  onboarding and WSO2 translation.

## Proposed Schemas

Inventory document:

```json
{
  "schemaVersion": "postman.spec.inventory.v1",
  "generatedAt": "2026-07-30T00:00:00.000Z",
  "scope": {
    "controller": "ado",
    "organization": "example-ado-org",
    "project": "Example Project"
  },
  "apis": []
}
```

Inventory API entry:

```json
{
  "key": "azure-apim:/subscriptions/.../service/apim/apis/drum-api",
  "providerType": "azure",
  "gatewayType": "azure-apim",
  "gatewayId": "/subscriptions/.../service/apim",
  "apiId": "/subscriptions/.../service/apim/apis/drum-api",
  "apiName": "Drum API",
  "apiVersion": "v1",
  "apiRevision": "1",
  "environment": "prod",
  "baseUrl": "https://example.azure-api.net",
  "basePath": "/drum",
  "spec": {
    "status": "exported",
    "format": "openapi-yaml",
    "path": "postman/spec-inventory/specs/drum-api.openapi.yaml",
    "sha256": "..."
  },
  "contractClass": "authoritative",
  "associationEvidence": [
    {
      "type": "gateway-tag",
      "key": "postman:repo",
      "value": "org/repo",
      "confidence": "strong"
    }
  ],
  "mappingConfidence": "strong"
}
```

Mapping document:

```json
{
  "schemaVersion": "postman.spec.mapping.v1",
  "entries": [
    {
      "approved": false,
      "inventoryKey": "azure-apim:/subscriptions/.../apis/drum-api",
      "source": {
        "providerType": "azure",
        "gatewayType": "azure-apim",
        "gatewayId": "/subscriptions/.../service/apim",
        "apiId": "/subscriptions/.../service/apim/apis/drum-api",
        "specArtifactPath": "postman/spec-inventory/specs/drum-api.openapi.yaml"
      },
      "target": {
        "ciProvider": "azure-devops",
        "organizationUrl": "https://dev.azure.com/example",
        "project": "Example Project",
        "repository": "Service-Repo",
        "branch": "main",
        "pipelineId": 159,
        "workspaceName": "Service API",
        "repoWriteMode": "branch"
      },
      "status": "active",
      "notes": ""
    }
  ]
}
```

Downstream source object:

```json
{
  "schemaVersion": "postman.spec.source.v1",
  "providerType": "azure",
  "gatewayType": "azure-apim",
  "gatewayId": "/subscriptions/.../service/apim",
  "apiId": "/subscriptions/.../service/apim/apis/drum-api",
  "specArtifactPath": "postman/spec-inventory/specs/drum-api.openapi.yaml",
  "expectedServiceName": "Drum API"
}
```

Provider-specific fields stay under `source.providerSpecific` when they cannot
be normalized.

## Provider Adapter Model

Add a gateway-neutral adapter layer above the current Azure provider registry:

```ts
interface InventoryProvider {
  readonly providerType: string;
  readonly gatewayType: string;
  probe(signal?: AbortSignal): Promise<ProviderProbeStatus>;
  enumerateGateways(signal?: AbortSignal): Promise<GatewayInventory[]>;
  enumerateApis(gateway: GatewayInventory, signal?: AbortSignal): Promise<ApiInventoryHeader[]>;
  hydrateApis(headers: ApiInventoryHeader[], signal?: AbortSignal): Promise<ApiInventory[]>;
  exportSpec(api: ApiInventory, signal?: AbortSignal): Promise<ExportedSpec | null>;
}
```

Azure APIM implementation can wrap the current `apim` provider. API Center can
wrap the current `api-center` provider. WSO2 would be a new adapter using WSO2
Publisher/Admin APIs if available.

The runtime contract should not require every provider to support every
capability. Each adapter declares:

- can enumerate APIs
- can export authored specs
- can synthesize partial specs
- can expose repo/source-control associations
- can expose environment/version/revision
- can expose service catalog ownership

Unsupported capability should produce explicit evidence, not an empty result.

## Mapping Confidence

Confidence levels:

| Level | Auto-dispatch allowed | Examples |
| --- | --- | --- |
| `exact` | Yes, after mapping file approval or committed binding | Existing mapping entry, `.postman/source-binding.yaml`, exact API id passed by caller |
| `strong` | No by default; can be configured to pre-approve only in mature estates | Gateway tag `postman:repo`, service catalog owner repo, API Center repo metadata |
| `medium` | No | IaC declaration, deployment pipeline metadata, source-control link on hosting resource |
| `weak` | No | Name, path, hostname, repository slug similarity |
| `manual` | No | Visible API with no usable ownership metadata |

Default behavior should be conservative:

- New entries start `approved: false`.
- Existing approved mappings stay approved when the source identity is stable.
- Removed APIs are marked `status: removed` and `approved: false`.
- Duplicate or conflicting evidence never selects automatically.

## Decision Matrix

| Context/trigger | Identity/tier | Credentials | Reads | Mutations | Persisted state | Outcome |
| --- | --- | --- | --- | --- | --- | --- |
| Controller inventory run | Subscription/resource-group/gateway scope | Gateway read credentials only | Enumerate, hydrate, export specs if supported | Writes local artifacts only | Inventory and proposed mapping | Reviewable artifacts |
| Controller mapping refresh | Stable inventory keys | No extra credentials | Existing mapping + new inventory | Writes mapping file only when configured | Preserves approvals | Approved state survives refresh |
| Approved dispatch | `approved: true`, target repo + branch + pipeline id | CI queue credential only | Mapping file | Queues target pipeline | Dispatch log | Target onboarding starts |
| Target onboarding | Exact `specSourceJson` or artifact path | Postman and repo credentials as needed | Source artifact or exact provider export | Postman assets and repo sync per target pipeline | Target repo `.postman` state | Service-specific assets updated |
| Ambiguous mapping | Weak/conflicting evidence | Read credentials only | Inventory evidence | None | Suggested mapping unapproved | Human review required |
| Missing spec | API visible but no exportable spec | Read credentials only | API metadata | None | Inventory entry with `spec.status=missing` | Manual or secondary derivation required |
| Partial spec | Provider can synthesize only partial contract | Read credentials only | Gateway metadata | Artifact write only | `contractClass=partial` | Dispatch only if approved |
| Retry after partial failure | Same stable keys | Same credentials | Refresh | Only missing artifacts rewritten | Existing approvals preserved | Converges or reports failure |

## State And Ownership

| State/asset | Owner | Source of truth | Version/migration | Corruption behavior | Deletion owner |
| --- | --- | --- | --- | --- | --- |
| Inventory JSON | Controller repo | Latest discovery run | `postman.spec.inventory.v1` | Regenerate from gateway | Controller workflow |
| Inventory Markdown | Controller repo | Generated from JSON | Same as JSON | Regenerate | Controller workflow |
| Exported specs | Controller repo artifact or committed inventory path | Gateway export at discovery time | Content hash | Re-export | Controller workflow |
| Mapping JSON | Controller repo | Human-approved file | `postman.spec.mapping.v1` | Fail closed; do not dispatch | Mapping owner |
| Dispatch log | Controller pipeline artifact | Dispatch run | Append/run-scoped | Retain failure evidence | Controller workflow |
| Target `.postman` files | Target repo | Target onboarding pipeline | Existing repo-sync contract | Existing target recovery | Target repo |

## Lifecycle

Creation:

- Controller inventory discovers APIs and emits unapproved mapping suggestions.
- Explicit source bindings or existing approved mappings may create exact entries.
- Target repositories and pipelines must already exist in v1.

Refresh and idempotency:

- Inventory refresh uses stable source keys and content hashes.
- Mapping merge preserves approved target fields when source identity remains the
  same.
- Changed spec content updates inventory artifacts but does not rewrite
  approvals.

Concurrent runs:

- Inventory runs should serialize per controller branch.
- Dispatch runs should serialize by target repo + branch.
- Target onboarding remains responsible for its own branch/repo-sync locking.

Promotion:

- Mapping approval is the promotion from suggested inventory to automation.
- Release channels should pin action/template versions before customer rollout.

Partial failure and rollback:

- Failed provider probes are recorded as scoped failures.
- Failed exact exports block that source entry.
- Failed dispatch does not unapprove the mapping.
- Rollback is reverting the mapping change or rerunning with the prior mapping
  artifact.

Retirement, deletion, restoration, and name reuse:

- Missing previously-seen APIs become `status: removed`.
- Removed entries are not deleted automatically.
- Restored APIs with the same stable provider identity can regain their prior
  mapping, but approvals should remain false if identity changed.
- Name reuse without matching provider identity creates a new unapproved entry.

## Security

Credential decision point:

- Inventory obtains only the provider credentials required for discovery/export.
- Dispatch obtains only the CI credential required to queue target pipelines.
- Target onboarding obtains Postman and repo credentials only after dispatch has
  selected a single approved target.

Identity and capability checks:

- Provider adapters distinguish "identity visible" from "export authorized".
- A source can be inventory-visible but not exportable.
- Exact export authorization failure is fatal for that entry.

Permission boundaries:

- Gateway read/export credentials must not allow mutation.
- CI dispatch credentials should queue only approved target pipelines.
- Target repo credentials remain scoped to the target repo.

Fork/untrusted contribution behavior:

- Pull requests from untrusted forks should not run live inventory or dispatch.
- Dry-run schema validation can run without cloud or CI queue credentials.

Failure-before-mutation guarantees:

- Mapping parse/schema validation happens before dispatch credentials are used.
- Dispatch validates all target identities before queueing any run when possible.
- No gateway mutation exists in the design.

## Implementation Plan

1. Add schemas and validators.
   - `postman.spec.inventory.v1`
   - `postman.spec.mapping.v1`
   - `postman.spec.source.v1`
   - Golden fixture tests for duplicate names, missing specs, partial specs, and
     removed APIs.

2. Add inventory builder.
   - Convert existing Azure `SpecCandidate` and `SpecExportResult` values into
     normalized `ApiInventory` entries.
   - Preserve provider-specific fields in a namespaced object.
   - Emit `inventory-json`, `mapping-json`, and artifact paths as new outputs.

3. Generalize estate mode.
   - Keep current `discover-estate` association roster for compatibility.
   - Add `inventory` or `discover-inventory` mode for full API inventory.
   - Do not remove existing `resolve-one`, `discover-many`, or `discover-estate`.

4. Add mapping merge.
   - Merge new inventory with existing mapping file.
   - Preserve approved target fields.
   - Add new APIs as unapproved suggestions.
   - Mark missing stable identities as removed and unapproved.

5. Add dispatch adapter interface.
   - Start with ADO queueing as an adapter.
   - Pass one `specSourceJson` object plus existing target pipeline fields.
   - Serialize by target repo + branch.
   - Record run ids and failures in a dispatch report.

6. Update ADO pipeline templates.
   - Accept `specSourceJson` or `specArtifactPath`.
   - Continue accepting APIM-specific parameters for compatibility.
   - Prefer exact artifact consumption over broad discovery in target repos.

7. Add optional WSO2 provider spike.
   - Determine whether the customer can expose WSO2 Publisher/Admin APIs.
   - Validate list APIs, export OpenAPI, version/environment fields, tags, and
     ownership metadata.
   - If management APIs are unavailable, classify WSO2 as association-only or
     manual-review for those services.

8. Add build-generated OpenAPI integration.
   - Define the `.csproj` / CI convention for emitting `swagger.json`.
   - Add a target-pipeline discovery step that resolves `swagger.json`,
     `swagger.yaml`, or `swagger.yml` from bounded generated-artifact locations.
   - Add a pre-promotion gate that fails when the generated spec is missing,
     invalid, or not attached as the source artifact for Postman/WSO2 consumers.
   - Document WSO2 translation as downstream packaging from the validated
     artifact, not as the primary contract source for active services.

9. Add live validation.
   - Azure APIM estate inventory.
   - Existing mapping approval.
   - ADO dispatch to a fixture target repo.
   - Target onboarding consuming `specSourceJson`.
   - Failure case for missing spec and duplicate ownership evidence.

## Customer Delivery Boundary

This design is larger than a single customer onboarding handoff. For a customer
pilot, the deliverable should stay narrow:

- validated target onboarding pipeline for the selected service;
- customer-ready reset/run instructions for that pipeline;
- known-good evidence from the target pipeline;
- optional discovery inventory output as a proof artifact, if the customer
  explicitly wants to inspect estate metadata.

The pilot should not promise full estate automation, automatic repository
mapping, WSO2 parity, or scaled dispatch unless those paths have been validated
with the customer's real gateway metadata and ADO topology.

Professional Services should have room to scale this into an operating program:

- establish metadata standards such as `postman:repo` or service catalog owner
  links;
- review and approve the initial mapping file;
- decide how repositories and pipelines are provisioned;
- phase rollout by business domain, gateway, or API criticality;
- define ownership for removed APIs, renamed services, and partial specs;
- train customer teams on maintaining mappings and source bindings.

For the current customer handoff, this proposal should be positioned as the
future-state architecture behind scale-out, not as a committed scope item for
the first service pipeline.

The near-term customer boundary is therefore:

- prove the selected service pipeline can consume a known OpenAPI artifact;
- document that active services should generate `swagger.json` in build;
- treat the gateway-discovery/backfill pipeline as a scale-out pattern for
  services that will not receive `.csproj` updates soon;
- avoid committing to org-wide nightly enforcement, repository provisioning, or
  WSO2 parity as part of the initial handoff.

## Contract And Release Propagation

| Repository/artifact | Contract change | Default/compatibility | Release dependency |
| --- | --- | --- | --- |
| `postman-azure-spec-discovery-action` | New inventory/mapping/source schemas and outputs | Additive; existing modes remain | New minor release |
| `postman-ado-spec-discovery` | Migrate APIM schemas to generic schemas or depend on shared core | Existing APIM params still accepted | Coordinated branch or package version |
| `postman-ado-pipeline-templates` | Discover generated `swagger.json/yaml`, then pass resolved path or `specSourceJson` | Existing `specPath` remains compatibility-only | Template version used by target repos |
| Application repositories | Emit build-generated `swagger.json` before promotion | New convention per framework; starts opt-in | Customer app-team adoption |
| WSO2 deployment pipeline | Translate validated OpenAPI into gateway contract | Gateway export remains backfill/drift signal | Customer gateway deployment path |
| Target customer repos | Optional source binding and generic spec input | Existing hardcoded spec path still works until migrated | Per-repo adoption |
| Documentation | Explain controller repo, mapping approval, dispatch, and caveats | New docs only | Published with release |

Migration:

- Ship generic schemas alongside current outputs.
- Add an APIM v1 mapping importer.
- Add build-generated OpenAPI as the preferred source type for active service
  repositories.
- Keep explicit `specPath` and APIM-specific dispatch parameters as
  compatibility aliases, but make generated-spec discovery the default for new
  rollout templates.
- Migrate one customer controller repo first.
- Remove APIM-specific aliases only after real adoption and release notes.

Rollback:

- Disable dispatch and keep inventory-only mode.
- Revert target templates to `spec-path`.
- Use prior approved mapping artifact.
- Keep existing `resolve-one` action behavior untouched.
- Let WSO2 continue consuming its existing gateway artifact while individual
  services adopt build-generated OpenAPI.

## Metadata Caveats

We are not sure the metadata we want will be present.

Expected metadata that may be absent or wrong:

- Gateway tags pointing to owning repositories.
- API Center ownership or repository links.
- WSO2 API labels, lifecycle metadata, or source repository references.
- App Service or Container Apps source-control records.
- IaC references in the target repository.
- Deployment pipeline outputs that identify the gateway API.
- API version, revision, or environment labels.
- Exportable OpenAPI definitions.
- Deterministic WSO2 export semantics.
- Framework-specific build hooks for `swagger.json` generation.

Design response:

- Treat metadata as evidence, not truth.
- Persist the evidence that led to a suggestion.
- Default suggested mappings to `approved: false`.
- Require explicit approval before dispatch.
- Label missing and partial specs clearly.
- Never claim deterministic mapping from weak name/path/host similarity.
- Prefer build-generated OpenAPI over gateway-exported OpenAPI for active
  services.
- Treat WSO2 export as backfill or drift evidence unless gateway export fidelity
  has been validated for that service.

This means the first estate run is likely an inventory and review exercise, not
a fully automated rollout. Automation becomes safer as customers add durable
metadata such as `postman:repo`, API Center ownership links, or committed source
bindings.

## Alternatives

| Option | Benefits | Risks/cost | Reversibility | Decision |
| --- | --- | --- | --- | --- |
| Extend existing Azure discovery action only | Fastest path for Azure/APIM | Azure-named repo accumulates non-Azure dispatch semantics | Medium | Accept only if generic contracts are clearly separated |
| Keep ADO spec discovery as the full controller | Already has mapping/dispatch proof | Remains APIM-specific and duplicates Azure provider work | Medium | Use as reference, not final abstraction |
| Extract shared discovery core | Cleanest long-term provider-neutral model | More package/release coordination | High | Preferred for WSO2 and non-Azure gateways |
| Caller-only pipeline changes | Minimal shared code | Pushes fragile discovery logic into every repo | Low | Reject |
| Do nothing | No implementation cost | Does not scale beyond bespoke APIM onboarding | High | Reject |

## Verification Plan

| Level | Scenario | Expected evidence | Status |
| --- | --- | --- | --- |
| Unit | Schema validation, mapping merge, confidence scoring | Golden fixture tests | Pending |
| Contract | Existing action inputs/outputs unchanged; new outputs additive | Action contract tests | Pending |
| Integration | Azure APIM inventory converted to generic inventory | Fixture and mocked Azure tests | Pending |
| Migration | Existing APIM mapping imports to generic mapping | Migration fixture tests | Pending |
| Failure/rollback | Missing spec, duplicate repo tags, removed API, dispatch queue failure | Failure fixtures and dispatch report | Pending |
| Representative live E2E | Controller inventory -> approved mapping -> ADO dispatch -> target onboarding | Live run ids and committed artifacts | Pending |

## Observability And Operations

Structured outputs:

- `inventory-json`
- `inventory-path`
- `mapping-json`
- `mapping-path`
- `dispatch-report-json`
- `dispatch-report-path`
- `changed-api-count`
- `approved-dispatch-count`
- `manual-review-count`

Audit/provenance:

- Record provider, gateway type, API id, version/revision, spec hash, evidence
  class, and run id.
- Do not log raw ARM ids except in explicit machine-readable outputs already
  intended for downstream automation.
- Do not serialize credentials, SAS query strings, private URLs, or full
  response bodies.

Dry-run and cleanup reporting:

- Inventory-only mode is the default.
- Mapping refresh can run without dispatch.
- Dispatch supports dry-run and reports the target runs it would queue.
- Cleanup is explicit: removed APIs are marked, not deleted.

Operational recovery:

- Re-run inventory to regenerate artifacts.
- Revert mapping file to roll back approvals.
- Re-run failed dispatch for approved entries.
- Use target pipeline logs for Postman asset-level failures.

## Risks And Conditions

| Risk/condition | Severity | Owner | Required resolution |
| --- | --- | --- | --- |
| Gateway metadata is missing or stale | High | Customer/API platform | Keep suggestions unapproved until reviewed |
| Exported specs do not exist for some live services | High | Provider adapter/customer | Mark missing; use manual spec authoring or separate derivation path |
| Weak matching wires wrong repo | High | Discovery core | Never auto-dispatch weak evidence |
| ADO dispatch permissions too broad | Medium | CI owner | Scope service connection and validate targets before queueing |
| Azure repo becomes non-Azure dumping ground | Medium | Maintainers | Extract shared core or keep non-Azure providers/adapters separate |
| Target pipelines still hardcode spec names | Medium | Template owners | Adopt `specSourceJson` / artifact path |
| Partial specs are mistaken for full contracts | Medium | Discovery core | Preserve `contractClass` and completeness labels |

## Readiness

- [x] Designed as a draft proposal
- [ ] Implemented
- [ ] Validated
- [ ] Released
- [ ] Adopted

## Review

Outcome: `REVISE`

Required conditions:

- Decide whether generic contracts live in this repository or a shared
  `spec-discovery-core` package.
- Validate which customer gateway metadata actually exists for APIM/WSO2.
- Confirm the target ADO template can consume `specSourceJson` without broad
  rediscovery.
- Run one representative live controller -> mapping -> dispatch flow before
  claiming validation.
