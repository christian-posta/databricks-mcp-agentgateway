```bash
source ~/bin/ai-keys
echo $DATABRICKS_PAT

databricks configure --token --host https://dbc-018eefb2-cdcc.cloud.databricks.com
```


Configure in VS Code:

```json
{
  "servers": {
    "databricks-unity-catalog": {
      "type": "stdio",
      "command": "uv",
      "args": [
        "--directory",
        "/Users/ceposta/python/databricks-mcp",
        "run",
        "unitycatalog-mcp",
        "-s",
        "samples.bakehouse",
        "-g",
        "01f0c63f9db919cb9411ea702ec4df93"
      ]
    }
  }
}
```

Make sure to grab the genie ID:

```
01f0c63f9db919cb9411ea702ec4df93
```

From Co-Pilot, prompt with:

```
Using the Genie tools, start a conversation about the bakehouse sales data
```

```
Using your tools from databricks-unity-catalog show me the first 10 rows from the sales_customers table in samples.bakehouse
```