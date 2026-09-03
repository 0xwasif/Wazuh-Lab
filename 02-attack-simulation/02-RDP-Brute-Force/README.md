
# RDP Brute Force Detection Lab — Wazuh

This lab simulates an **RDP brute-force attack** against a Windows endpoint and implements custom Wazuh detection rules with automated firewall response.

**MITRE ATT&CK:** `T1110.001 — Password Guessing`

## 🖥️ Setup

* **Wazuh Manager** monitoring Windows endpoint `win_ep_01`
* Windows attacker performing repeated RDP login attempts
* **Windows Event Logs + Sysmon** forwarded to Wazuh
* Custom Wazuh rules for RDP brute-force detection
* Windows Firewall used for automated IP blocking

## ⚔️ Attack & Detection

The attacker repeatedly attempted to authenticate to the Windows endpoint using invalid credentials.

Two custom rules were used:

| Rule     | Purpose                                                           |
| -------- | ----------------------------------------------------------------- |
| `100070` | Detects NTLM RDP network logon failures                           |
| `100071` | Detects **8 failures within 120 seconds from the same source IP** |

The detection rule is based on:

```text
Event 4625
    ↓
Logon Type 3 / NTLM
    ↓
Rule 100070
    ↓
8 failures / 120 seconds
    ↓
Rule 100071
    ↓
RDP Brute Force Detected
```

### Important Field

Windows stores the source IP in:

```text
win.eventdata.ipAddress
```

Therefore, the correlation rule uses:

```xml
<same_field>win.eventdata.ipAddress</same_field>
```

instead of `same_srcip`.

## 🛡️ Active Response

When rule `100071` fires, Wazuh triggers an active response using `netsh`.

```xml
<active-response>
  <disabled>no</disabled>
  <command>netsh</command>
  <location>local</location>
  <rules_id>100071</rules_id>
  <timeout>120</timeout>
</active-response>
```

### Response Flow

```text
RDP Brute Force
      ↓
Wazuh Rule 100071
      ↓
Active Response
      ↓
netsh
      ↓
Windows Firewall
      ↓
Attacker IP Blocked
      ↓
Block Removed After 120s
```

## 🔗 Alert Timeline

The Wazuh dashboard showed the complete sequence:

```text
Failed RDP Logons
      ↓
Rule 100070
      ↓
Rule 100071
      ↓
Account Lockout
      ↓
Active Response
      ↓
Firewall IP Block
      ↓
Block Removed
```

Windows also independently locked the targeted account, providing a second layer of defense.

## ✅ Result

The lab successfully demonstrated:

* RDP brute-force simulation
* Windows Event `4625` analysis
* Custom Wazuh rule creation
* Source-IP correlation
* MITRE ATT&CK mapping
* Automated active response
* Windows Firewall IP blocking
* Account-lockout protection
* End-to-end alert investigation

The attacker IP was automatically blocked at the firewall level **without manual analyst intervention**.

## 📸 Evidence

|  # | Screenshot                                                                              | What it shows                                      |
| -: | --------------------------------------------------------------------------------------- | -------------------------------------------------- |
|  1 | [Account Lockout](https://claude.ai/chat/screenshots/account-lockout.png)               | Account lockout after repeated failed RDP attempts |
|  2 | [RDP Connection Refused](https://claude.ai/chat/screenshots/rdp-connection-refused.png) | RDP connection refused after account lockout       |
|  3 | [Active Response Config](https://claude.ai/chat/screenshots/active-response-config.png) | `netsh` active response configuration              |
|  4 | [Detection Rules](https://claude.ai/chat/screenshots/detection-rules.png)               | Custom rules `100070` and `100071`                 |
|  5 | [Alert Timeline](https://claude.ai/chat/screenshots/alert-timeline.png)                 | Wazuh alert chain from detection to response       |
|  6 | [Firewall Blocked IP](https://claude.ai/chat/screenshots/firewall-blocked-ip.png)       | Windows Firewall blocking `192.168.100.12`         |
