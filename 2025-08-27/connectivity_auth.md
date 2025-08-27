# Connectivity and Authentication for SNMP

This document supports the "Ingesting SNMP Data 101" webinar, detailing how to configure connectivity and authentication for SNMP versions 1, 2c, and 3 using Logstash's SNMP input plugin. It also covers testing connectivity with `snmpwalk`, troubleshooting with `stdout`/`rubydebug`, and using `@metadata` fields. This section builds on the basics from [snmp_basics.md](./snmp_basics.md) and prepares for the live demo in [live_demo.md](./live_demo.md).

## SNMP Versions and Authentication

SNMP supports three versions with different authentication mechanisms:

- **v1**: Basic protocol with minimal security, using a community string (like a password).
- **v2c**: Enhanced features (e.g., bulk requests), still uses community strings.
- **v3**: Secure, with user-based authentication (USM) supporting encryption and stronger credentials.

We'll configure Logstash to connect to a device (e.g., a switch or simulator like [snmpsim](https://github.com/etingof/snmpsim)) using these versions. Configurations require numeric OIDs—use tools like `snmptranslate` or [Observium MIB Browser](https://mibs.observium.org/mib/) to find them.

## Step 1: Testing Connectivity with `snmpwalk`

Before configuring Logstash, verify device connectivity using `snmpwalk` (install via `apt install snmp` or equivalent).

If `snmpwalk` fails, check IP/port, community string, or firewall settings.

### v1/v2c Examples

#### With OID

```bash
snmpwalk -v2c -c public 127.0.0.1:1161 1.3.6.1.2.1.1.1.0
```

- `-v2c`: Specifies SNMP v2c.
- `-c public`: Community string (replace with your device's string).
- `127.0.0.1:1161`: Device IP and port (adjust for your setup).
- `1.3.6.1.2.1.1.1.0`: OID for system description.

Expected output: `iso.3.6.1.2.1.1.1.0 = STRING: "Generic Switch Model XYZ"`.

#### With MIB Name (for Human-Readable Testing)

```bash
snmpwalk -v2c -c public 127.0.0.1:1161 -m SNMPv2-MIB sysDescr.0
```

- `-m SNMPv2-MIB`: Loads the MIB for name-to-OID translation (note: Logstash requires numeric OIDs in configs).

Expected output: `SNMPv2-MIB::sysDescr.0 = STRING: "Generic Switch Model XYZ"`.

### v3 Example

Including this for those eager for more, though we won't use SNMPv3 in the 101 session.

```bash
snmpwalk -v3 -l authPriv -u snmpuser -a SHA -A authpass -x AES -X privpass 127.0.0.1:1161 1.3.6.1.2.1.1.1.0
```

- `-l authPriv`: Authentication and privacy (encryption).
- `-u snmpuser`: Username.
- `-a SHA`: Authentication protocol (SHA or MD5).
- `-A authpass`: Authentication password.
- `-x AES`: Encryption protocol (AES or DES).
- `-X privpass`: Encryption password.

Expected output: `iso.3.6.1.2.1.1.1.0 = STRING: "Generic Switch Model XYZ"`.

## Step 2: Troubleshooting with `stdout` and `rubydebug`

For basic testing, use the `stdout` output with `rubydebug` codec to inspect events, including `@metadata` fields (e.g., `host_address`, `host_port`, `host_community`, `host_protocol`). This helps verify connectivity and data structure.

Example config for testing:

```ruby
input {
  snmp {
    hosts => [{host => "udp:127.0.0.1/1161", community => "public"}]
    get => ["1.3.6.1.2.1.1.1.0"]
    target => "snmp"
    ecs_compatibility => "v8"
    oid_mapping_format => "ruby_snmp"
    interval => 60
  }
}
output {
  stdout { codec => rubydebug }
}
```

- Run Logstash and check output for fields like `[snmp][SNMPv2-MIB::sysDescr.0]` and `@metadata` contents (e.g., `@metadata[host_address]`).
- Use this to confirm OIDs, field naming, and connectivity before full ingestion.

## Step 3: Logstash Configuration

Configure the SNMP input plugin to poll a device. Below are examples for each version, using the setup from [live_demo.md](./live_demo.md). Always set `target => "snmp"` to namespace data (no default). Enable ECS with `ecs_compatibility => "v8"` or globally in `logstash.yml` (`pipeline.ecs_compatibility: v8`).

### v1/v2c Configuration

```ruby
input {
  snmp {
    hosts => [{host => "udp:127.0.0.1/1161", community => "public"}]
    get => ["1.3.6.1.2.1.1.1.0"]  # Numeric OID for sysDescr.0
    target => "snmp"
    ecs_compatibility => "v8"
    oid_mapping_format => "ruby_snmp"
    interval => 60
  }
}
```

- `host`: UDP endpoint (IP:port).
- `community`: Community string (e.g., `public`).
- `oid_mapping_format`: Controls output field names (e.g., `ruby_snmp` for `SNMPv2-MIB::sysDescr.0`).

### v3 Configuration

```ruby
input {
  snmp {
    hosts => [{
      host => "udp:127.0.0.1/1161",
      security_name => "snmpuser",
      security_level => "authPriv",
      auth_protocol => "SHA",
      auth_password => "authpass",
      priv_protocol => "AES",
      priv_password => "privpass"
    }]
    get => ["1.3.6.1.2.1.1.1.0"]
    target => "snmp"
    ecs_compatibility => "v8"
    oid_mapping_format => "ruby_snmp"
    interval => 60
  }
}
```

- `security_name`: Username for v3.
- `security_level`: `authPriv` (auth + encryption), `authNoPriv` (auth only), or `noAuthNoPriv`.
- `auth_protocol`/`auth_password`: For authentication.

Since these passphrases are more secure than the community string `public`, we'll cover using Logstash's secure keystore to protect sensitive keys in "Ingesting SNMP Traps 201". For this session, we'll stick to SNMP v1/v2c examples.

## Next Steps

- Test connectivity with `snmpwalk` and numeric OIDs.
- Use `stdout`/`rubydebug` to inspect `@metadata` and event structure.
- See [live_demo.md](./live_demo.md) for a full example.
- Check [mibs_formatting.md](./mibs_formatting.md) for data formatting.
