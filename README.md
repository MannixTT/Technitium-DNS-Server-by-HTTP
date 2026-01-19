# Technitium DNS Server by HTTP

## Overview
This template allows you to monitor individual **Technitium DNS Server** nodes or clusters via the HTTP API. It provides comprehensive insights into DNS query statistics, response codes, cluster health, DHCP scopes, and security-related parameters.

## Requirements
* **Zabbix Version**: 7.4 or higher.
* **Technitium API Token**: A read-only API token with appropriate permissions for Settings, Dashboard, DHCP, and Administration.

## Tested versions
* **Technitium DNS**: 14.3.
* **Zabbix**: 7.4.

## Configuration
To set up this template, follow the principle of least privilege:
1. **Create a User**: In Technitium (`Administration > Users`), create a new user with read-only rights for the required sections.
2. **Generate Token**: Log in with the created user and generate an API key via the top right drop-down menu (`Create API token`).
3. **Assign Macros**: Assign the template to the host and configure the macros `{$TECHNITIUM.URL}` and `{$TECHNITIUM.TOKEN}`.

## Setup
1. Import the `technitium_dns_server_by_http.yaml` template into Zabbix.
2. Link the template to your Technitium DNS host.
3. Ensure the macro `{$TECHNITIUM.STATS.TYPE}` matches your desired observation period (default is `LastDay`).

---

## Macros used
| Name | Description | Default |
| :--- | :--- | :--- |
| `{$TECHNITIUM.URL}` | The Technitium API endpoint URL (`<scheme>://<host/fqdn>:<port>`). | `None` |
| `{$TECHNITIUM.TOKEN}` | The API token created for a read-only user. | `None` |
| `{$TECHNITIUM.STATS.TYPE}` | Period for dashboard statistics (LastHour, LastDay, LastWeek, LastMonth, LastYear). | `LastDay` |
| `{$TECHNITIUM.DOMAIN.MAX}` | Threshold for high query rate warning per domain. | `600` |
| `{$TECHNITIUM.DOMAIN.BLOCK.MAX}` | Threshold for warning on hits by a blocked domain. | `600` |
| `{$TECHNITIUM.DOMAIN.BLOCK.RATE.MIN}` | Minimum % of blocked queries before the problem triggers for low blocking percentage | `15` |
| `{$TECHNITIUM.LLD.FILTER.DOMAIN.MATCHES}` | Regex filter for top queried domains. | `.*` |

---

# Technitium DNS Server: Static Items and Triggers

This document lists all non-LLD (static) metrics and alerts configured in the Technitium DNS Server by HTTP template.

## Items

| Name | Description | Type | Key |
| :--- | :--- | :--- | :--- |
| **Get metrics** | The master script item that fetches all data from the Technitium API. | SCRIPT | `technitium.get.metrics` |
| **Get stats** | Raw JSON response containing dashboard statistics. | DEPENDENT | `technitium.get.stats` |
| **Get settings** | Raw JSON response containing server settings. | DEPENDENT | `technitium.get.settings` |
| **Get update** | Raw JSON response regarding software updates. | DEPENDENT | `technitium.get.update` |
| **Get DHCP** | Raw JSON response containing DHCP scope lists. | DEPENDENT | `technitium.get.dhcp` |
| **Get errors** | Captures and stores any errors returned during API requests. | DEPENDENT | `technitium.get.errors` |
| **Total queries** | Total number of DNS queries received in the selected period. | DEPENDENT | `technitium.dns.queries.total` |
| **Total clients** | Total number of unique clients that made requests. | DEPENDENT | `technitium.clients.total` |
| **Queries blocked in %** | A calculated item showing the percentage of blocked queries vs. total queries. | CALCULATED | `technitium.dns.queries.blocked` |
| **Blocking Status** | Indicates if the global DNS blocking feature is enabled (True/False). | DEPENDENT | `technitium.settings.blocklist.state` |
| **DNS Forwarder Protocol** | The protocol used for upstream forwarding (e.g., Udp, Tcp, Quic, Https). | DEPENDENT | `technitium.settings.dns.forwarder_protocol` |
| **Installed Version** | The current software version of the Technitium DNS Server. | DEPENDENT | `technitium.update.installed_version` |
| **Available Version** | The latest version available for update. | DEPENDENT | `technitium.update.available_version` |
| **Update available** | Boolean indicator if a software update is pending. | DEPENDENT | `technitium.update.available` |
| **Cluster Domain** | The domain name configured for the Technitium cluster. | DEPENDENT | `technitium.cluster.domain` |
| **DNS Zones** | Number of DNS zones configured on the server. | DEPENDENT | `technitium.dns.zones` |
| **Hits served by Cache** | Number of DNS queries answered from the local cache. | DEPENDENT | `technitium.dns.cached_entries` |
| **Domains on Blocklist** | Total number of domains currently on the server's blocklist. | DEPENDENT | `technitium.dns.blocklist.blocked_domains` |
| **Domains on Allowlist** | Total number of domains currently on the server's allowlist. | DEPENDENT | `technitium.dns.blocklist.allowed_domains` |
| **Response Code: No Error** | Count of responses with RCODE 0 (NOERROR). | DEPENDENT | `technitium.dns.response.code[no_error]` |
| **Response Code: NXDOMAIN** | Count of responses with RCODE 3 (NXDOMAIN). | DEPENDENT | `technitium.dns.response.code[nxdomain]` |
| **Response Code: Refused** | Count of responses with RCODE 5 (REFUSED). | DEPENDENT | `technitium.dns.response.code[refused]` |
| **Response Code: Server Failure** | Count of responses with RCODE 2 (SERVFAIL). | DEPENDENT | `technitium.dns.response.code[srvfail]` |

---

## Triggers

| Name | Severity | Description | Expression |
| :--- | :--- | :--- | :--- |
| **Unusual high NXDOMAIN rate** | HIGH | Fired when the query rate is high (>1/s) and over 60% of queries result in NXDOMAIN. | `rate(...)>1 and (rate(NXDOMAIN)/rate(Total))>0.6` |
| **Blocking is disabled** | AVERAGE | Alert if the global blocking feature is turned off. | `last(.../technitium.settings.blocklist.state)<>"true"` |
| **Unencrypted forwarder protocol in use** | HIGH | Alert if DNS forwarder protocol is set to unencrypted UDP or TCP. | `last(.../technitium.settings.dns.forwarder_protocoll)="Udp" or "Tcp"` |
| **There are errors in requests to API** | HIGH | Triggered if the API returns an error message. | `length(last(.../technitium.get.errors))>0` |
| **Technitium Update available** | WARNING | Fired if a new version of Technitium is available for installation. | `last(.../technitium.update.available)<>"false"` |
| **Technitium Version has changed** | INFO | Informational alert when the software version is updated or downgraded. | `last(.../technitium.update.installed_version,#1)<>last(...,#2)` |
| **REFUSED DNS Response Code detected** | HIGH | Alert if the server starts refusing a high volume of DNS requests. | `rate(.../technitium.dns.reponse.code[refused],5m)>1` |
| **Server Failure DNS Response Code detected** | HIGH | Alert if the server or upstream is returning SERVFAIL. | `rate(.../technitium.dns.reponse.code[srvfail],5m)>1` |
| **Low percentage of blocked queries** | WARNING | Alert if the percentage of blocked queries falls below 15%. | `last(.../technitium.dns.queries.blocked)<{$TECHNITIUM.DOMAIN.BLOCK.RATE.MIN}` |

## LLD rules
| Name | Description | Type | Key and additional info |
| :--- | :--- | :--- | :--- |
| Cluster Discovery | Discovers nodes within the Technitium cluster. | DEPENDENT | `technitium.cluster.discovery` |
| DHCP Scopes Discovery | Discovers configured DHCP address pools. | DEPENDENT | `technitium.dhcp.scopes.discovery` |
| Forwarder Discovery | Discovers upstream DNS forwarders. | DEPENDENT | `technitium.dns.forwarder.discovery` |
| Query Response Discovery | Stats for response types (Authoritative, Cached, etc.). | DEPENDENT | `technitium.dns.query_responses.discovery` |
| Query Types Discovery | Stats for record types (A, AAAA, MX, etc.). | DEPENDENT | `technitium.dns.query_types.discovery` |
| Top Blocked Discovery | Monitors the most frequently blocked domains. | DEPENDENT | `technitium.dns.top_blocked_domains.discovery` |
| Top Clients Discovery | Monitors the most active DNS clients. | DEPENDENT | `technitium.dns.top_clients.discovery` |
| Top Domains Discovery | Monitors the most frequently queried domains. | DEPENDENT | `technitium.dns.top_domains.discovery` |
| Blocklist Discovery | Monitors configured blocklist URLs. | DEPENDENT | `technitium.settings.blocklist_urls.discovery` |

---

## Discovery Rules Details

This document provides a detailed breakdown of all Low-Level Discovery (LLD) rules, including their item and trigger prototypes, used for dynamic monitoring of the Technitium DNS Server.

### 1. Cluster Discovery
Discovers individual nodes within a Technitium cluster to monitor their connectivity and health.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.cluster.discovery` |
| **Item Prototype 1** | **Name**: `Cluster Node {#NODE.NAME} ({#NODE.TYPE}): State` |
| | **Key**: `technitium.cluster.node.state[{#NODE.NAME}]` |
| **Item Prototype 2** | **Name**: `Cluster Node {#NODE.NAME} ({#NODE.TYPE}): Uptime` |
| | **Key**: `technitium.cluster.node.uptime[{#NODE.NAME}]` |
| **Trigger Prototype 1**| **Name**: `Technitium: Cluster Node {#NODE.NAME} is not connected` |
| | **Expression**: `last(...)<>"Self" and last(...)<>"Connected"` |
| | **Severity**: AVERAGE |
| **Trigger Prototype 2**| **Name**: `Cluster Node {#NODE.NAME} was restarted` |
| | **Expression**: `last(...) < 10m` |
| | **Severity**: INFO |

---

### 2. DHCP Scopes Discovery
Automatically identifies and monitors DHCP address pools configured on the server.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dhcp.scopes.discovery` |
| **Item Prototype 1** | **Name**: `DHCP Scope {#SCOPE_NAME}: Status` |
| | **Key**: `technitium.dhcp.scope.enabled[{#SCOPE_NAME}]` |
| **Item Prototype 2** | **Name**: `DHCP Scope {#SCOPE_NAME}: Address Range` |
| | **Key**: `technitium.dhcp.scope.range[{#SCOPE_NAME}]` |
| **Trigger Prototype** | **Name**: `DHCP Scope {#SCOPE_NAME} is disabled` |
| | **Expression**: `last(...)<>"enabled"` |
| | **Severity**: AVERAGE |

---

### 3. Forwarder Discovery
Discovers configured upstream DNS forwarders to ensure they are present in the configuration.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dns.forwarder.discovery` |
| **Item Prototype** | **Name**: `Forwarder: {#FORWARDER.NAME}` |
| | **Key**: `technitium.dns.forwarder[{#FORWARDER.NAME}]` |

---

### 4. DNS Query Response Discovery
Provides metrics for different response categories such as Authoritative, Cached, or Blocked.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dns.query_responses.discovery` |
| **Item Prototype** | **Name**: `DNS Response: {#RESPONSE_TYPE}` |
| | **Key**: `technitium.dns.queries.response[{#RESPONSE_TYPE}]` |

---

### 5. DNS Query Types Discovery
Tracks query volume for specific DNS record types like A, AAAA, MX, and CNAME.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dns.query_types.discovery` |
| **Item Prototype** | **Name**: `DNS Query Type: {#QUERY_TYPE}` |
| | **Key**: `technitium.dns.query_type[{#QUERY_TYPE}]` |

---

### 6. Top Blocked Domains Discovery
Identifies domains that are hitting blocklists most frequently.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dns.top_blocked_domains.discovery` |
| **Item Prototype** | **Name**: `Blocked Domain: {#DOMAIN_NAME} Hits` |
| | **Key**: `technitium.dns.blocked.hits[{#DOMAIN_NAME}]` |
| **Trigger Prototype** | **Name**: `Domain {#DOMAIN_NAME} is being blocked excessively` |
| | **Expression**: `last(...) > {$TECHNITIUM.DOMAIN.BLOCK.MAX}` |
| | **Severity**: AVERAGE |

---

### 7. Top Clients Discovery
Monitors the most active DNS clients and their rate-limiting status.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dns.top_clients.discovery` |
| **Item Prototype 1** | **Name**: `Client: {#CLIENT_LABEL} ({#CLIENT_IP}) Hits` |
| | **Key**: `technitium.dns.client.hits[{#CLIENT_IP}]` |
| **Item Prototype 2** | **Name**: `Client: {#CLIENT_LABEL} rate limit status` |
| | **Key**: `technitium.dns.client.rate_limited[{#CLIENT_IP}]` |
| **Trigger Prototype** | **Name**: `Client {#CLIENT_LABEL} is being rate limited` |
| | **Expression**: `last(...)=1` |
| | **Severity**: AVERAGE |

---

### 8. Top Domains Discovery
Identifies the most frequently queried legitimate domains.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.dns.top_domains.discovery` |
| **Item Prototype** | **Name**: `Domain: {#DOMAIN_NAME} Hits` |
| | **Key**: `technitium.dns.domain.hits[{#DOMAIN_NAME}]` |
| **Trigger Prototype** | **Name**: `High query rate for domain {#DOMAIN_NAME}` |
| | **Expression**: `last(...) > {$TECHNITIUM.DOMAIN.MAX}` |
| | **Severity**: WARNING |

---

### 9. Blocklist Discovery
Monitors the availability of external blocklist URLs used by the server.

| Component | Detail |
| :--- | :--- |
| **LLD Rule Key** | `technitium.settings.blocklist_urls.discovery` |
| **Item Prototype** | **Name**: `Blocklist: {#BLOCKLIST_URL}` |
| | **Key**: `technitium.settings.blocklist.url[{#BLOCKLIST_URL}]` |
| **Trigger Prototype** | **Name**: `Blocklist removed: {#BLOCKLIST_URL}` |
| | **Expression**: `nodata(...,7h)=1` |
| | **Severity**: WARNING |

---

## Feedback and Contribution
Please report issues or suggest improvements on the GitHub Issues page. Pull requests for feature enhancements are welcome!

**Official Repository:** [https://github.com/MannixTT/Technitium-DNS-Server-by-HTTP](https://github.com/MannixTT/Technitium-DNS-Server-by-HTTP)
