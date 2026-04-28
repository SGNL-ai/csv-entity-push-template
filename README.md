# CSV Entity Push

This repository is an example of how to configure GitHub and GitHub Actions for managing change control for CSV files used by SGNL. As changes to CSV files under `csv_files/` are made, those changes are pushed via entity push to SGNL.

## Repository Structure

```
├── .github/
│   └── workflows/
│       ├── csv-entity-push.yml     # Pushes CSV changes to SGNL on merge to main
│       └── validate-csv-ids.yml    # Generates missing IDs and validates on pull request
├── csv_sor_example/
│   └── csv_sor_example.yaml        # Example SGNL SOR configuration for entity push
└── csv_files/                      # CSV files for each entity (add your CSVs here)
```

Both workflows use the reusable [csv-entity-push-action](https://github.com/SGNL-ai/csv-entity-push-action) GitHub Action.

## Getting Started

### 1. Create a new repository from this template

Click **"Use this template"** → **"Create a new repository"** on the [template repo page](https://github.com/SGNL-ai/csv-entity-push-template). Then clone your new repository:

```sh
git clone <your-repo-url>
cd <your-repo-name>
```

### 2. Create a new SOR in your SGNL client

Use the YAML under `csv_sor_example/csv_sor_example.yaml` as a reference to create a new SOR in your SGNL client. Update the entities with the entity names for your use case, and update the attributes with the column names for each CSV.

### 3. Download CSV templates from SGNL

Once the SOR is created, select **Import CSV** and download the CSV template from SGNL for each entity. Make sure each entity CSV has an `id` column. The `id` values will be populated automatically when a pull request is opened.

### 4. Configure the GitHub environment

Create a GitHub environment called **`sgnl client`** in the repository settings (Settings > Environments > New environment), then add the following variables and secrets to that environment:

**Variables:**

| Variable | Description |
|---|---|
| `ENTITY_PUSH_ISS` | Issuer value for the event |
| `ENTITY_PUSH_AUD` | Audience value for the event |
| `ENTITY_PUSH_URL` | HTTP endpoint to POST events to |

**Secrets:**

| Secret | Description |
|---|---|
| `ENTITY_PUSH_TOKEN` | Bearer token for endpoint authentication |

### 5. Add your CSV data

Add each entity's CSV to the `csv_files/` directory and populate it with your data, leaving the `id` values blank. CSV files follow the naming pattern `SOR Name-Entity Name.csv`.

| Filename | Entity Name |
|---|---|
| `SOR Name-Entity1.csv` | `Entity1` |
| `SOR Name-Entity2.csv` | `Entity2` |

On pull request, any empty `id` fields will automatically be populated with a unique GUID. On merge to `main`, each change (add, update, remove) will be pushed to the corresponding entity in your SGNL client.

## How It Works

### GitHub Action: Validate CSV IDs

On pull requests targeting `main`, the validate workflow first generates UUIDs for any rows with empty `id` values and commits them back to the PR branch. It then validates that every row has a non-empty `id`. If any rows are still missing IDs, the check fails and the PR is blocked from merging.

**Example** — given a CSV with an empty `id` field:

```
attribute1,attribute2,id
value1,value2,
```

After the workflow runs, the missing ID is populated and committed:

```
attribute1,attribute2,id
value1,value2,2f437b78-7b00-4bc2-b8f3-64471aa5c4a9
```

Rows that already have an `id` value are left unchanged.

### GitHub Action: CSV Entity Push

When CSV files in `csv_files/` are merged to `main`, the [csv-entity-push-action](https://github.com/SGNL-ai/csv-entity-push-action) detects row-level changes and sends SCIM SET events to SGNL:

- **New row** — `urn:ietf:params:SCIM:event:prov:create:full`
- **Modified row** — `urn:ietf:params:SCIM:event:prov:put:full`
- **Deleted row** — `urn:ietf:params:SCIM:event:prov:delete`

Events are JWT-encoded (HS256, `secevent+jwt`) and POSTed to the configured endpoint.

**Event payload (before JWT encoding):**

```json
{
  "iss": "<configured issuer>",
  "iat": 1745524800,
  "jti": "<unique uuid>",
  "aud": "<configured audience>",
  "sub_id": {
    "format": "scim",
    "uri": "/<entity_name>/<id>",
    "externalId": "id",
    "id": "<row id>"
  },
  "events": {
    "<event_type>": {
      "data": {
        "schemas": ["urn:ietf:params:scim:schemas:core:2.0:<entity_name>"],
        "attribute1": "value1",
        "attribute2": "value2"
      }
    }
  }
}
```

The `id` column is excluded from the `data` section since it is already present in `sub_id`. All other columns are included as key-value pairs.
