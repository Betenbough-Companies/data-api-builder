# Betenbough Data API Builder

This repository contains a forked version of Azure Data API Builder configured to expose the BetenboughDW database via REST and GraphQL APIs. The API runs on Azure Web App: `production-betenbough-dataapi-laqdtbvq5xlpc`.

## 🎯 Quick Overview

**What is this?** A REST and GraphQL API that exposes database views from the BetenboughDW database without writing code.

**When do I use this?** When you need to expose new database views/tables to external consumers via API endpoints.

**Do I need to deploy code?** Usually NO. The code deployment is only needed if the upstream Data API Builder repository has bug fixes we need to merge. Configuration changes are done directly in Azure.

---

## 🚀 How to Add a New Entity (Table/View) to the API

### Step 1: Create the Database View

1. Create your view in your **local** or **dev/test** `BetenboughDW` database
2. **CRITICAL**: The view MUST be in the `[data-api]` schema
3. Note the EXACT case of your view name and all column names (case-sensitivity matters!)

Example:
```sql
CREATE VIEW [data-api].[MyNewEntity] AS
SELECT 
    EntityId,
    EntityName,
    CreatedDate
FROM SomeSourceTable
```

> **Important:** You're developing against a local or dev/test BetenboughDW database. Production changes will be applied later in Step 5.

### Step 2: Set Up Local Development Environment

1. **Copy the config template to create your local config:**
   ```powershell
   cd src\Service
   Copy-Item dab-config.jsontemplate dab-config.json
   ```
   
2. **Edit `dab-config.json` and update the connection string** (line 5) to point to your local or development database:
   ```json
   "connection-string": "YOUR_LOCAL_CONNECTION_STRING_HERE"
   ```
   
   > **Note:** The `dab-config.json` file is git-ignored, so your local credentials won't be committed.

### Step 3: Add Your Entity to the Configuration

Edit your local `dab-config.json` and add your new entity to the `"entities"` section.

#### ⚠️ CRITICAL: Case Sensitivity Rules

- **Database object names** (tables, views, columns) are **case-sensitive**
- They MUST match EXACTLY as defined in the database
- Entity names in the config should also follow the same casing

#### Example Entity Configuration:

```json
"entities": {
  "MyNewEntity": {
    "source": {
      "object": "data-api.MyNewEntity",
      "type": "table",
      "key-fields": [ "EntityId" ]
    },
    "graphql": {
      "enabled": true,
      "type": {
        "singular": "MyNewEntity",
        "plural": "MyNewEntities"
      }
    },
    "rest": { "enabled": true },
    "permissions": [
      {
        "role": "anonymous",
        "actions": [ { "action": "*" } ]
      }
    ],
    "relationships": {
      "RelatedEntity": {
        "cardinality": "one",
        "target.entity": "RelatedEntityName",
        "source.fields": [ "RelatedEntityId" ],
        "target.fields": [ "RelatedEntityId" ]
      }
    }
  }
}
```

#### Understanding the Configuration:

- **`source.object`**: Must be `"data-api.YourViewName"` (schema.object format)
- **`source.type`**: Use `"table"` for both tables and views
- **`key-fields`**: The primary key or unique identifier column(s)
- **`graphql.type.singular`**: The name used in GraphQL for a single item
- **`graphql.type.plural`**: The name used in GraphQL for collections
- **`permissions`**: Set to `"anonymous"` with `"*"` actions for public access
- **`relationships`**: Optional - defines how this entity relates to others

### Step 4: Test Locally

1. **Run the API locally:**
   ```powershell
   cd src\Service
   dotnet run
   ```

2. **Test with Postman:**
   
   **REST API:**
   - GET all: `http://localhost:5000/api/MyNewEntity`
   - GET by ID: `http://localhost:5000/api/MyNewEntity/EntityId/123`
   - Filter: `http://localhost:5000/api/MyNewEntity?$filter=EntityName eq 'Test'`

   **GraphQL API:**
   - Endpoint: `http://localhost:5000/graphql`
   - Example Query:
     ```graphql
     {
       myNewEntities {
         entityId
         entityName
         createdDate
       }
     }
     ```

3. **Verify case sensitivity:**
   - Test different casing in your requests to ensure everything matches exactly
   - If you get "entity not found" errors, double-check your casing

### Step 5: Deploy to Production

Once your local testing is successful:

1. **Update the template file for future developers:**
   - Copy ONLY the `"entities"` section from your `dab-config.json`
   - Paste it into `src\Service\dab-config.jsontemplate` (replacing the existing `"entities"` section)
   - Commit and push this change to the repository

2. **⚠️ CRITICAL: Apply changes to BOTH production databases:**
   
   The production API reads from `BetenboughDW_DWSnapshot`, which is refreshed nightly from `BetenboughDW`. To avoid breaking the API, you must update BOTH databases:

   **a) Update BetenboughDW (source database):**
   - Execute your `CREATE VIEW` script on the production `BetenboughDW` database
   - Ensure it's in the `[data-api]` schema
   - This ensures the view persists after the nightly refresh

   **b) Update BetenboughDW_DWSnapshot (API database):**
   - Execute the SAME `CREATE VIEW` script on the `BetenboughDW_DWSnapshot` database
   - Ensure it's in the `[data-api]` schema
   - This makes the view immediately available to the API

   > **Warning:** If you only update BetenboughDW, the production API will break until the next nightly refresh! Always update both databases.

3. **Update the Azure Web App configuration:**
   - Go to Azure Portal → `production-betenbough-dataapi-laqdtbvq5xlpc` Web App
   - In the left menu, find **Development Tools** → **App Service Editor (Preview)**
   - Navigate to `dab-config.json`
   - Replace the `"entities"` section with your updated entities section
   - **Save the file**

4. **Restart the Web App:**
   - Go back to the Web App's Overview page
   - Click the **Restart** button at the top
   - Wait for the restart to complete (~30 seconds)

5. **Test in Production:**
   - REST: `https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/api/MyNewEntity`
   - GraphQL: `https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/graphql`

---

## 🛠️ Prerequisites

Before you start, ensure you have:

- [ ] **.NET 8.0 SDK** installed ([Download](https://dotnet.microsoft.com/download/dotnet/8.0))
- [ ] **Visual Studio 2022** or **VS Code** (optional, but helpful)
- [ ] **Access to BetenboughDW database** (to create views and test)
- [ ] **Access to Azure Portal** (to edit Web App configuration)
- [ ] **Postman** or similar API testing tool ([Download](https://www.postman.com/downloads/))
- [ ] **SQL Server Management Studio** or **Azure Data Studio** (for database work)

---

## 📁 Repository Structure

```
src/Service/
├── dab-config.jsontemplate   ← Template config (source-controlled)
└── dab-config.json           ← Your local config (git-ignored)

.github/workflows/
└── production-betenbough-dataapi-appserviceplan-laqdtbvq5xlpc.yml   ← CI/CD pipeline
```

---

## 🔄 When Does Code Deployment Happen?
�️ Understanding the Database Structure

**Two Production Databases:**
- **BetenboughDW** - The source database where changes should be made
- **BetenboughDW_DWSnapshot** - A nightly snapshot that the API actually queries

**The Nightly Refresh:**
- Every night, `BetenboughDW_DWSnapshot` is refreshed from `BetenboughDW`
- Any views/tables in `BetenboughDW` will automatically appear in the snapshot after the refresh

**Why Update Both Databases?**
- If you only update `BetenboughDW`, the API won't see your changes until tomorrow's refresh
- If you only update `BetenboughDW_DWSnapshot`, your changes will be overwritten tonight
- **Always update both** to ensure immediate availability and persistence

---

## 🐛 Common Issues & Troubleshooting

### Issue: Production API broken after deployment

**Cause:** View was only created in `BetenboughDW`, not in `BetenboughDW_DWSnapshot`

**Solution:**
- Execute your `CREATE VIEW` script on `BetenboughDW_DWSnapshot` immediately
- The API will work as soon as the Web App is restarted
- Remember: always update BOTH databases going forward

### Issue: "Entity not found" or API returns 404

**Cause:** Case sensitivity mismatch

**Solution:**
- Verify the view exists in the `[data-api]` schema in `BetenboughDW_DWSnapshot`re)
- For typical entity additions/modifications, you **DO NOT** need to trigger a deployment

---

## 🐛 Common Issues & Troubleshooting

### Issue: "Entity not found" or API returns 404

**Cause:** Case sensitivity mismatch

**Solution:**
- Verify the view exists in the `[data-api]` schema
- Check that `source.object` matches EXACTLY: `"data-api.ViewName"`
- Confirm column names in `key-fields` match the database exactly (case-sensitive)

### Issue: "Column does not exist" errors

**Cause:** Column name casing doesn't match database

**Solution:**
- Query the database to see exact column names:
  ```sql
  SELECT COLUMN_NAME 
  FROM INFORMATION_SCHEMA.COLUMNS 
  WHERE TABLE_SCHEMA = 'data-api' 
    AND TABLE_NAME = 'YourViewName'
  ```
- Update your config to match exactly

### Issue: Changes not reflected after restart

**Cause:** Browser cache or Web App didn't restart properly

**Solution:**
- Clear browser cache or use Postman (no cache)
- In Azure Portal, check the Web App logs to confirm restart
- Wait 30-60 seconds after restart before testing

### Issue: Local development connection fails

**Cause:** Connection string authentication method

**Solution:**
- For local development, you may need to use SQL authentication instead of `ActiveDirectoryInteractive`
- Example local connection string:
  ```
  Data Source=localhost;Initial Catalog=BetenboughDW;User ID=yourusername;Password=yourpassword;
  ```

### Issue: Relationship not working in GraphQL

**Cause:** Incorrect field mapping or cardinality

**Solution:**
- Verify `source.fields` and `target.fields` column names are correct (case-sensitive!)
- Ensure the target entity exists in the config
- Check cardinality: use `"one"` for foreign key to primary key, `"many"` for one-to-many

---

## 📖 API Usage Examples

### REST API

The REST API is available at `/api/EntityName`

**Get all entities:**
```http
GET https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/api/Homesites
```

**Get by primary key:**
```http
GET https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/api/Homesites/HomesiteId/12345
```

**Filter with OData:**
```http
GET https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/api/Homesites?$filter=PhaseId eq 100
```

**Select specific fields:**
```http
GET https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/api/Homesites?$select=HomesiteId,Address
```

**Pagination:**
```http
GET https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/api/Homesites?$top=10&$skip=20
```

### GraphQL API

The GraphQL endpoint is at `/graphql`

**Endpoint:** 
```
https://production-betenbough-dataapi-laqdtbvq5xlpc.azurewebsites.net/graphql
```

**Example Query:**
```graphql
{
  homesites(filter: { phaseId: { eq: 100 } }) {
    items {
      homesiteId
      address
      phase {
        phaseId
        phaseName
      }
      floorPlan {
        floorPlanId
        planName
      }
    }
  }
}
```

**With Relationships:**
```graphql
{
  homesites {
    items {
      homesiteId
      tasks {
        items {
          taskId
          taskName
        }
      }
    }
  }
}
```

---

## 🔐 Security & Permissions

**Development:**
- [ ] Create view in `[data-api]` schema in local/dev BetenboughDW database
- [ ] Note exact casing of view and column names
- [ ] Copy `dab-config.jsontemplate` to `dab-config.json` locally
- [ ] Update local connection string in `dab-config.json`
- [ ] Add entity configuration to local `dab-config.json`
- [ ] Test locally with `dotnet run` from `src\Service`
- [ ] Verify REST endpoint works in Postman
- [ ] Verify GraphQL query works (if applicable)

**Source Control:**
- [ ] Copy `entities` section back to `dab-config.jsontemplate`
- [ ] Commit and push `dab-config.jsontemplate` changes

**Production Deployment:**
- [ ] ⚠️ Execute `CREATE VIEW` on production `BetenboughDW` database
- [ ] ⚠️ Execute SAME `CREATE VIEW` on production `BetenboughDW_DWSnapshot` database
This allows unrestricted read/write access. If you need to restrict access in the future, consult the [Data API Builder permissions documentation](https://learn.microsoft.com/en-us/azure/data-api-builder/authorization).

---

## 🤝 Getting Help

- **Azure Data API Builder Documentation:** https://learn.microsoft.com/en-us/azure/data-api-builder/
- **OData Query Syntax:** https://www.odata.org/documentation/
- **GraphQL Documentation:** https://graphql.org/learn/

---

## 📝 Quick Reference Checklist

When adding a new entity, follow this checklist:

- [ ] Create view in `[data-api]` schema in production database
- [ ] Note exact casing of view and column names
- [ ] Copy `dab-config.jsontemplate` to `dab-config.json` locally
- [ ] Update local connection string in `dab-config.json`
- [ ] Add entity configuration to local `dab-config.json`
- [ ] Test locally with `dotnet run` from `src\Service`
- [ ] Verify REST endpoint works in Postman
- [ ] Verify GraphQL query works (if applicable)
- [ ] Copy `entities` section back to `dab-config.jsontemplate`
- [ ] Commit and push `dab-config.jsontemplate` changes
- [ ] Update `dab-config.json` in Azure App Service Editor
- [ ] Restart Azure Web App
- [ ] Test production endpoints

---

## ⚙️ Why is This Forked?

This repository is forked from the official Azure Data API Builder because:
- The upstream repository doesn't include deployment artifacts for Azure Web App
- We needed a solution that deploys as a standard .NET application to Azure App Service
- The fork allows us to maintain our custom CI/CD pipeline

The deployment pipeline handles building and publishing the application, but configuration management remains manual through the Azure Portal's App Service Editor.
