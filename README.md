# sf-d360-branches-datakit

A small, self-contained **Data 360 (Data Cloud) DevOps data kit** in Salesforce DX source format. It
contains:

- a **File Upload (CSV) data stream** → **DLO** `branch_offices_seedcsv` → mapped to a custom
  **DMO** `Branch Office` (`Branch_Office__dlm`)
- the `UploadedFiles` connection it depends on
- a 50-row fictional seed file: [`data/branch-offices-seed.csv`](data/branch-offices-seed.csv)

The kit was built in a sandbox as **Data Kit Type = DevOps** (`dataKitType = SANDBOX`) in the
**`default`** data space. It was retrieved by following Salesforce's
[Deploy a DevOps Data Kit Using the CLI](https://developer.salesforce.com/docs/data/data-cloud-dev/guide/dc-deploy_data_kit_using_cli.html).

> **DevOps data kits can't be packaged.** Package Manager lets you add one under *Data Package Kit
> Definition*, but **Upload** fails with *"DevOps Data Kit Type is not supported for packaging"*.
> The supported ways to move this kit are the Metadata API / `sf` CLI (below) or change sets.

## Contents

| Path | What |
|---|---|
| `manifest/package.xml` | The kit's manifest: 33 members across 12 metadata types |
| `force-app/main/default/dataPackageKitDefinitions/` | The kit itself (`branch_offices_datakit`) |
| `force-app/main/default/DataPackageKitObjects/` | Kit membership records |
| `force-app/main/default/dataSourceBundleDefinitions/`, `dataStreamTemplates/`, `dataSourceObjects/` | The data stream bundle and its DLO |
| `force-app/main/default/dataKitObjectTemplates/`, `dataKitObjectDependencies/`, `mktDataSources/` | The `UploadedFiles` connection pieces |
| `force-app/main/default/objects/Branch_Office__dlm/` | The custom DMO and its fields |
| `force-app/main/default/dataSrcDataModelFieldMaps/` | DLO → DMO field mappings |
| `data/branch-offices-seed.csv` | Seed data (`OfficeCode,OfficeName,Region,State,Country,OpenedDate`) |

**Modifications from the raw retrieve:**
1. The Key Qualifier field `objects/Branch_Office__dlm/fields/KQ_OfficeCode__c.field-meta.xml` was
   **deleted**, as the Salesforce doc instructs (step 6). It isn't deployable.
2. Source-org identifiers (user name, user ID, connection record ID) in the `UploadedFiles` template
   and the data stream template were **replaced with placeholders** (`Redacted User`,
   `005000000000000AAA`, `9cg000000000000AAA`). The redacted source still validates: a check-only
   deploy succeeded with **33/33 components, 0 errors**.

## Prerequisites (target org)

- **Data Cloud provisioned** in the target. For a sandbox, Data Cloud must be provisioned in its
  parent production org.
- A data space named **`default`** that is **ACTIVE**. Use the same data-space prefix as the source.
- Salesforce CLI (`sf`) installed.
- **File Upload connector:** in **Data Cloud Setup → Other Connectors**, set **UploadedFiles** to
  **Active**. It's inactive by default in sandboxes.

## Deploy

```bash
# 1. Authorize the target org (sandbox shown; use login.salesforce.com for production)
sf org login web --alias target --instance-url https://test.salesforce.com

# 2. Validate first (check-only, changes nothing)
sf project deploy start --manifest manifest/package.xml --target-org target --dry-run

# 3. Deploy
sf project deploy start --manifest manifest/package.xml --target-org target
```

You can also monitor the deploy in **Setup → Deployment Status**.

## After the deploy (manual steps in the target)

Only the kit *definition* moves as metadata. Data Cloud's runtime wiring has to be done by hand
in each target:

1. **Connector:** confirm **UploadedFiles** is **Active**. Reauthorize or reactivate it if it shows
   Inactive, then redeploy if components didn't link.
2. **Data Cloud Setup → Data Kits → `branch offices datakit`:** confirm the stream bundle, data model,
   and connection are listed.
3. **Create or activate the data stream** from the kit, then **upload `data/branch-offices-seed.csv`**
   to ingest the 50 rows.
4. **Verify:** in **Data Explorer**, open the `Branch Office` DMO in the `default` data space and
   check that the rows appear.

## Troubleshooting (from the Salesforce doc)

| Error | Fix |
|---|---|
| `DMO relationships require a FieldSrcTrgtRelationship…` | Delete the XML files named in the error and redeploy. |
| `no FieldSrcTrgtRelationship named … found` | Add the named members under `FieldSrcTrgtRelationship` in `package.xml`, retrieve again, and delete any `KQ_*` files. |
| `Unable to update the DLO: <fields>` | Source and target DLO schemas differ. Sync them before deploying. |
| Connectors inactive after deploy | Reauthorize them in **Data Cloud → Connectors**, then redeploy. |
| `KQ_*` field rejected | Delete it: `cd force-app/main/default/objects && find . -name 'KQ_*' -exec rm {} \;` |

## Re-retrieving from a source org

```bash
# Get the kit's manifest. In the UI it's Data Kits → (kit) → Download Manifest.
# Via REST:
sf api request rest "services/data/v64.0/ssot/datakit/branch_offices_datakit/manifest" --target-org source
# ...convert dataKitMembers[] into manifest/package.xml, then:
sf project retrieve start --manifest manifest/package.xml --target-org source
cd force-app/main/default/objects && find . -name 'KQ_*' -exec rm {} \;
```
