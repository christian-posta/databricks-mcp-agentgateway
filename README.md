```bash
source ~/bin/ai-keys
echo $DATABRICKS_PAT

databricks configure --token --host https://dbc-f1002050-7c6a.cloud.databricks.com
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
        "01f0c67c0cda1eb5b34096d638d3be44"
      ]
    }
  }
}
```

Make sure to grab the genie ID:

```
01f0c67c0cda1eb5b34096d638d3be44
```

From Co-Pilot, prompt with:

```
Can you list available Genies?
```

```
Using the Genie tools, start a conversation about the bakehouse sales data
```

```
Using your tools from databricks-unity-catalog show me the first 10 rows from the sales_customers table in samples.bakehouse
```