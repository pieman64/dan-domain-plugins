# dan-domain-plugins

A Claude Code marketplace of domain investing and research plugins.

## Install

```bash
/plugin marketplace add <owner>/<repo>
/plugin install comp-domain-research@dan-domain-plugins
```

Replace `<owner>/<repo>` with the GitHub repository this marketplace is published from.

## Plugins

### comp-domain-research

Turn one reported domain sale into a ranked shortlist of 10 unregistered names that follow the same syntax.

Given a sale ("LocalFix.com sold on GoDaddy for $3,226"), it decomposes the syntax, sweeps 120 to 200 candidates for availability, conflict-checks the top 10 against company and trademark records, builds a market-comps table from live asking prices, suggests tiered pricing, and writes a report.

**Requires the Unstoppable Domains MCP server.** Availability is the whole product, so the skill refuses to guess. Connect it with:

```bash
claude mcp add --transport http unstoppabledomains https://api.unstoppabledomains.com/mcp/v1
```

Then run `/mcp` in an interactive session and complete the browser sign-in. The skill verifies the `ud_domains_search` tool responds before it starts a sweep, and stops with setup instructions if it does not.

Accepted fallbacks, if the MCP server is unavailable, are the Unstoppable marketplace UI (only when an enabled add-to-cart control and a registration price are visible) and RDAP status codes. Both are recorded in the report, and neither supplies the asking-price comps.

## License

MIT
