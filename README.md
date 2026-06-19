# Splunk DBX Add-on for ClickHouse

A custom JDBC driver Add-on that enables [Splunk DB Connect](https://splunkbase.splunk.com/app/2686) to connect to [ClickHouse](https://clickhouse.com/) databases, including ClickHouse Cloud.

Since Splunk does not officially provide a ClickHouse driver Add-on, this package bundles the necessary JDBC driver and configuration to register ClickHouse as a supported database type in both **Splunk Enterprise** and **Splunk Cloud**.

## Quick Start

1. Download the latest `.tgz` from [Releases](../../releases) (or use the file in this repo)
2. Install via Splunk Web → **Manage Apps** → **Install app from file**
3. Open **DB Connect** → **Configuration** → **Drivers** → Click **Reload**
4. Verify "ClickHouse" appears with a green checkmark ✅

## Documentation

| Document | EN | JA |
|----------|----|----|
| **README** | [README.md](Splunk_JDBC_clickhouse/README.md) | [README_ja.md](Splunk_JDBC_clickhouse/README_ja.md) |
| **Setup Guide** | [SETUP_GUIDE.md](Splunk_JDBC_clickhouse/SETUP_GUIDE.md) | [SETUP_GUIDE_ja.md](Splunk_JDBC_clickhouse/SETUP_GUIDE_ja.md) |
| **Technical Note** | [TECHNICAL_NOTE.md](Splunk_JDBC_clickhouse/TECHNICAL_NOTE.md) | [TECHNICAL_NOTE_ja.md](Splunk_JDBC_clickhouse/TECHNICAL_NOTE_ja.md) |

## Compatibility

| Component | Version |
|-----------|---------|
| Splunk Enterprise | 9.x, 10.x |
| Splunk Cloud | ✅ Supported |
| Splunk DB Connect | 4.x |
| ClickHouse JDBC Driver | 0.9.8 (shaded fat JAR) |
| Default Port | `443` (SSL) |

> **Note**: This Add-on defaults to port `443` for maximum compatibility. Splunk Cloud blocks outbound traffic on non-standard ports (e.g., `8443`) by default. ClickHouse Cloud accepts HTTPS on both `443` and `8443`. See the [Technical Note](Splunk_JDBC_clickhouse/TECHNICAL_NOTE.md#d-splunk-cloud-port-compatibility) for details.

## License

This Add-on packages the official [ClickHouse JDBC Driver](https://github.com/ClickHouse/clickhouse-java) which is licensed under the Apache License 2.0.
