# Streamkap

The `streamkap` MCP server manages real-time CDC (change data capture) pipelines in the user's Streamkap tenant: sources, destinations, pipelines, transforms, Kafka topics, schemas, and alerts.

- Start with `streamkap_health_overview` for tenant-wide health, or `streamkap_diagnose_pipeline` for one pipeline.
- Prefer the `*_brief` list tools for overviews, then `get_*` tools for detail.
- Pass `response_format: "text"` for compact summaries.
- Use `streamkap_search_docs` to turn a connector error into the docs page that explains it.
- Create, update, deploy, pause, and restart change live infrastructure: confirm with the user first.

The first call opens a browser sign-in to the user's Streamkap account. Full docs: https://docs.streamkap.com/mcp-server
