# n8n-Cloudflare-DDNS-Updater

**TL;DR**: [n8n](https://n8n.io) Workflows for AI agent-aided IP updaters for [Cloudflare](https://cloudflare.com) DDNS. The issue I am trying to solve is that existing updaters tend to just update specific record/subdomains that are manually set, not all records/subdomains in a Cloudflare Zone. With these Workflows, a lightweight AI agent determines and updates all.

**The problem:** Traditional DDNS tools (ddclient, cloudflare-ddns scripts, the Home Assistant Cloudflare integration, the DDNS updater in UniFi Network etc.) require you to hardcode every domain and subdomain you want updated. Add a new subdomain in Cloudflare? You have to remember to update your DDNS config too (and in some cases like the HA integration completely wipe your setup and start over).

**This solution:** An AI agent dynamically discovers _all_ zones and A records accessible by the Cloudflare API token it is given. Add new domains or subdomains in Cloudflare and they're automatically covered — zero config changes needed.

**Cost:** Despite using an LLM, the AI agent only fires when the IP actually changes (not every check interval). Claude tells me that with the default gpt-4.1-nano at $0.10/1M input tokens, each execution costs roughly **$0.0019**.

## Workflow Variants

| Variant | File | IP Source | Use Case |
|---------|------|-----------|----------|
| **Local IP** | `cloudflare-ddns-local.json` | [ipify.org](https://www.ipify.org) | n8n runs locally on the server/network whose IP you want to track |
| **UniFi Site Manager** | `cloudflare-ddns-unifi.json` | [UniFi Site Manager API](https://developer.ui.com) | Monitor a remote UniFi gateway's WAN IP from anywhere |
| **SSH Remote IP** | `cloudflare-ddns-ssh.json` | SSH + curl | SSH into a remote device to check its public IP |
| **Webhook** | `cloudflare-ddns-webhook.json` | Incoming HTTP POST | Remote device pushes its IP to n8n (near-instant updates) |

## How It Works

All four variants share the same downstream logic — they only differ in how the IP is obtained:

```
  Get IP (varies by variant)
        │
        ▼
   Compare IPs ─────────────── Uses n8n workflow static data
        │                       to remember the last known IP
        ▼
   IP Changed?
      │       │
     No      Yes
      │       │
    Stop      ▼
          AI Agent (LangChain)
            ├── Tool: List Zones
            ├── Tool: List DNS Records
            └── Tool: Update DNS Record
                    │
                    ▼
              Summary of changes
```

**IP source per variant:**
- **Local** — Schedule Trigger (every 5 min) → ipify.org HTTP request
- **UniFi** — Schedule Trigger (every 5 min) → UniFi Site Manager API → extract WAN IP by host ID
- **SSH** — Schedule Trigger (every 5 min) → SSH into remote device → run `curl ipify.org`
- **Webhook** — Webhook Trigger (on-demand POST) → validate payload → extract IP from body

**Downstream flow (shared by all):**

1. **Compare IPs** - A Code node using `$getWorkflowStaticData('global')` to persist the last known IP between runs. On the very first run it seeds the IP and triggers the agent in audit mode (see Safety below).
2. **IP Changed?** - An IF node that gates everything downstream. When the IP hasn't changed (the vast majority of runs), the workflow stops here. No Cloudflare API calls, no LLM costs.
3. **AI Agent** - A LangChain Tools Agent (gpt-4.1-nano, temperature 0) with three HTTP Request tools that call the Cloudflare API. It lists all zones, lists all A records per zone, identifies records matching the old IP, patches them to the new IP, and reports a summary.

## Prerequisites

- A running **n8n instance** (self-hosted or, if not using the local WAN IP version, n8n Cloud)
- A **Cloudflare API token** with these permissions:
  - `Zone : Zone : Read`
  - `Zone : DNS : Edit`
  - Zone Resources: "All zones" (or specific zones)
- An **OpenAI API key** (or any LLM provider with tool-calling support - see [Using a Different LLM](#using-a-different-llm))
- _(UniFi variant only)_ A **UniFi Site Manager API key** from [unifi.ui.com](https://unifi.ui.com)
- _(SSH variant only)_ **SSH access** to a remote device with `curl` installed. **SSH must work independently of DNS** - if the device's IP changes and DNS is stale, hostname-based SSH will fail. Use a static IP, VPN (Tailscale, WireGuard), or persistent tunnel as the SSH host
- _(Webhook variant only)_ A **remote device** capable of making HTTP POST requests (any Linux box, router with scripting, etc.)

## Quick Start

### Local IP Variant

1. Download `cloudflare-ddns-local.json`
2. In n8n: **Workflows** → **Import from File** → select the JSON
3. **OpenAI credential:**
   - Click the **OpenAI Chat Model** node → Credential dropdown → **Create New Credential** → **OpenAI API**
   - Paste your OpenAI API key → Save
4. **Cloudflare credential:**
   - Click any of the 3 tool nodes (List Zones, List DNS Records, Update DNS Record)
   - Credential dropdown → **Create New Credential** → **Header Auth**
   - **Name:** `Authorization`
   - **Value:** `Bearer YOUR_CLOUDFLARE_API_TOKEN`
   - Save, then select this same credential on the other two tool nodes
5. Toggle **Active** (top-right switch)

### UniFi Site Manager Variant

1. Download `cloudflare-ddns-unifi.json`
2. In n8n: **Workflows** → **Import from File** → select the JSON
3. **UniFi credential:**
   - Click the **Fetch UniFi Hosts** node → Credential dropdown → **Create New Credential** → **Header Auth**
   - **Name:** `X-API-KEY`
   - **Value:** your UniFi Site Manager API key
   - Save
4. **Find your Host ID:**
   - Click the **Fetch UniFi Hosts** node → **Test step**
   - Find your gateway device in the output → copy its `id` field
5. **Configure Host ID:**
   - Open the **Extract WAN IP** Code node
   - Replace `YOUR_HOST_ID_HERE` with your actual host ID
6. **OpenAI credential:** same as local variant step 3
7. **Cloudflare credential:** same as local variant step 4
8. Toggle **Active**

### SSH Remote IP Variant

1. Download `cloudflare-ddns-ssh.json`
2. In n8n: **Workflows** → **Import from File** → select the JSON
3. **SSH credential:**
   - Click the **SSH Remote IP Check** node → Credential dropdown → **Create New Credential** → **SSH**
   - **Host:** IP address or hostname of the remote device
   - **Port:** 22 (default)
   - **Username:** your SSH username
   - **Authentication:** Password or Private Key
   - Save
4. **OpenAI credential:** same as local variant step 3
5. **Cloudflare credential:** same as local variant step 4
6. Toggle **Active**

> **Important:** SSH access must work independently of DNS. If the remote device's IP changes and DNS hasn't updated yet, hostname-based SSH will fail. Use a static IP, VPN (e.g. Tailscale, WireGuard), or tunnel as the SSH host.

### Webhook Variant

1. Download `cloudflare-ddns-webhook.json`
2. In n8n: **Workflows** → **Import from File** → select the JSON
3. **Webhook authentication:**
   - Click the **Webhook Trigger** node → Credential dropdown → **Create New Credential** → **Header Auth**
   - Set a **Name** and **Value** as your shared secret (e.g. Name: `X-Secret`, Value: `my-secret-token-123`)
   - Save
4. **OpenAI credential:** same as local variant step 3
5. **Cloudflare credential:** same as local variant step 4
6. Toggle **Active** — your webhook URL is now: `https://YOUR_N8N_DOMAIN/webhook/ddns-update`
7. **Set up the remote device** — add a cron job or script that sends:
   ```bash
   curl -X POST https://YOUR_N8N_DOMAIN/webhook/ddns-update \
     -H "Content-Type: application/json" \
     -H "X-Secret: my-secret-token-123" \
     -d '{"ip": "'$(curl -s https://api.ipify.org)'"}'
   ```
   For example, as a cron job running every 5 minutes:
   `*/5 * * * * /path/to/ddns-webhook.sh`

## Testing

You can test the schedule-based variants (Local, UniFi, SSH) without waiting for a real IP change:

1. Open the **Compare IPs** Code node
2. Temporarily change this line:
   ```javascript
   const previousIp = staticData.lastKnownIp || null;
   ```
   to:
   ```javascript
   const previousIp = "1.2.3.4";
   ```
3. Click **Test Workflow** - the AI agent will fire, discover all your zones and A records, find no records matching `1.2.3.4`, and report that no updates were needed. This confirms the full pipeline works end-to-end.
4. **Revert the change** before activating the workflow for production use.

For the **Webhook variant**, send a test POST with a fake IP instead:
```bash
curl -X POST https://YOUR_N8N_DOMAIN/webhook-test/ddns-update \
  -H "Content-Type: application/json" \
  -H "X-Secret: my-secret-token-123" \
  -d '{"ip": "1.2.3.4"}'
```
(Use `/webhook-test/` while the workflow execution window is open in n8n.)

> **Note:** n8n's workflow static data (`$getWorkflowStaticData`) only persists when the workflow runs via its trigger (i.e., when active). Manual test executions do not save static data - this is expected n8n behavior, not a bug.

## Safety

- **Match-only updates** - Only updates A records whose current content exactly matches the previous known IP. Records intentionally pointing to different IPs are never touched.
- **No create/delete** - The agent is instructed to only update existing records. It will not create new DNS records or delete existing ones.
- **First-run audit** - On the very first run the agent lists all zones and A records and reports what it found, but does NOT update anything. This lets you verify the setup is correct before any DNS changes are made. Updates begin from the second run onward.
- **Max iterations** - The agent is capped at 100 tool-calling iterations to prevent runaway API calls. This is high enough to handle accounts with many zones and records.
- **Webhook authentication** - The webhook variant requires a shared secret header, rejecting unauthorized requests.
- **Webhook IP validation** - The webhook variant rejects private/reserved IP addresses (RFC1918, CGNAT, loopback, link-local) to prevent non-public IPs from being written into DNS records.

## Cost Estimate

| Scenario | IP Changes / Month | Est. Cost / Month |
|----------|-------------------|-------------------|
| Stable residential IP | 2–4 | < $0.01 |
| Frequent dynamic IP | ~30 | ~$0.06 |
| Very unstable | ~100 | ~$0.19 |

Each AI execution uses approximately 10,000–15,000 input tokens and 1,000–2,000 output tokens with gpt-4.1-nano ($0.10/$0.40 per 1M tokens). The 5-minute schedule checks that don't trigger the AI cost nothing.

## Configuration

| Parameter | Where | Default | Description |
|-----------|-------|---------|-------------|
| Check interval | Schedule Trigger node | 5 minutes | How often to check for IP changes (schedule-based variants) |
| LLM model | OpenAI Chat Model node | `gpt-4.1-nano` | Which model powers the agent |
| Temperature | OpenAI Chat Model node | `0` | LLM randomness (0 = deterministic) |
| Max iterations | Cloudflare DNS Agent node → Options | `100` | Max tool-calling rounds |
| Host ID | Extract WAN IP Code node | _(must configure)_ | UniFi variant only |
| SSH command | SSH Remote IP Check node | `curl -s https://api.ipify.org?format=json` | SSH variant only — command run on remote device |
| Webhook path | Webhook Trigger node | `ddns-update` | Webhook variant only — the URL path |

## Agent System Prompt

The AI agent's behaviour is governed by the following system prompt (identical across all four variants). This is reproduced here so you can audit the safety boundaries without importing the JSON:

> You are a Cloudflare DNS management assistant. Your job is to update DNS A records when a public IP address changes.
>
> You have access to the Cloudflare API via an HTTP request tool. The API base URL is `https://api.cloudflare.com/client/v4`. Authentication is already configured — just make requests.
>
> **Workflow:**
> 1. GET `/zones` to list all zones. Extract the zone id and name from each result.
> 2. For each zone, GET `/zones/{zone_id}/dns_records?type=A` to list all A records. Extract record id, name, content, and zone_id.
> 3. Filter the A records to find those where the content field matches the OLD IP address exactly.
> 4. For each matching record, PATCH `/zones/{zone_id}/dns_records/{record_id}` with `{"content": "NEW_IP"}`.
> 5. After completing all updates, provide a clear summary of what was changed.
>
> **Safety rules:**
> - ONLY update A records whose content exactly matches the OLD IP. Never modify records with different IPs.
> - Never delete any records.
> - Never create new records.
> - If no records match the old IP, report that no updates were needed.
>
> The Cloudflare API returns paginated results. If `total_pages > 1`, make additional requests with `?page=2`, `?page=3`, etc.

On the first run, the agent receives an audit-only prompt instead: it lists all zones and A records, flags any that don't match the current IP, but makes no changes.

## Using a Different LLM

The OpenAI Chat Model sub-node can be swapped for any LangChain-compatible LLM node in n8n that supports tool calling:

1. Delete the **OpenAI Chat Model** node
2. Add your preferred LLM node (Anthropic Claude, Google Gemini, Ollama, etc.)
3. Connect it to the **Cloudflare DNS Agent** node's `ai_languageModel` input
4. Configure its credentials

The agent's system prompt and tools remain unchanged — they work with any model that supports function/tool calling.

## Known Issues & Troubleshooting

**"Invalid URL" error on tool nodes**
The HTTP Request Tool nodes use `placeholderDefinitions` with `{placeholder}` syntax instead of `$fromAI()` expressions. This is intentional - `$fromAI()` does not work reliably with `toolHttpRequest` v1.1 ([n8n#14274](https://github.com/n8n-io/n8n/issues/14274)).

**Why not use the built-in Cloudflare node?**
n8n's built-in Cloudflare node only supports SSL certificate operations, not DNS record CRUD. The workflow uses direct HTTP requests to the Cloudflare API instead.

**Agent says "no records updated" during testing**
This is expected when testing with a fake IP like `1.2.3.4`. The agent correctly identifies that no A records match that IP and reports no changes needed. This confirms the safety mechanism works.

**Static data resets between manual test runs**
`$getWorkflowStaticData('global')` only persists during active (triggered) runs. Manual "Test Workflow" executions don't save static data. Activate the workflow for persistent state.

**Credential fields appear empty after import**
n8n never embeds credential values (API keys, tokens) in exported workflow JSON. You must set up credentials manually after importing - see the setup sticky notes inside each workflow.

**SSH variant: DNS-independent access required**
See Prerequisites above. The SSH host must be reachable without relying on the DNS records this workflow updates.

**Webhook variant: n8n must be reachable from the remote device**
The remote device needs to be able to reach your n8n instance's webhook URL. If using HTTPS (recommended), ensure your TLS certificate is valid. The webhook path can be customized in the Webhook Trigger node.

**No IPv6 (AAAA) record support**
Only A records (IPv4) are targeted. AAAA records for IPv6 are not currently updated. This could be added by including `type=AAAA` in the DNS record queries and extending the agent's instructions.

## License

MIT — see [LICENSE](LICENSE) for details.
