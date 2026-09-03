# SSH Brute Force Detection Lab — Wazuh

This lab simulates an **SSH brute-force attack** against an Ubuntu endpoint and implements a custom Wazuh detection rule with an automated firewall response.

**MITRE ATT&CK:** `T1110.001 — Password Guessing`

## 🖥️ Setup

* **Wazuh Manager** monitoring an Ubuntu endpoint
* Kali Linux used as the attacker
* SSH authentication logs forwarded to Wazuh
* Custom Wazuh rule for brute-force detection
* `iptables` used for automated IP blocking

## ⚔️ Attack & Detection

The attacker repeatedly attempted to authenticate to the Ubuntu endpoint using invalid SSH credentials.

A custom rule was created to detect **5 failed SSH logins from the same source IP within 60 seconds**.

```text
Kali Linux
    ↓
Failed SSH Logons
    ↓
Rule 5760
    ↓
5 failures / 60 seconds
    ↓
Rule 100621
    ↓
SSH Brute Force Detected
```

### Custom Rule

```xml
<group name="ssh,syslog,authentication_failed,">
  <rule id="100621" level="10" frequency="5" timeframe="60">
    <if_matched_sid>5760</if_matched_sid>
    <same_srcip />
    <description>
      Multiple SSH Login failure: 5 failed logins from $(srcip) within 1 minute
    </description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
    <group>authentication_failures,brute_force,</group>
  </rule>
</group>
```

The rule uses:

```xml
<same_srcip />
```

to ensure that the failures originate from the **same attacker IP**.

## 🛡️ Active Response

When rule `100621` fires, Wazuh triggers the `firewall-drop` active response.

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100621</rules_id>
  <timeout>60</timeout>
</active-response>
```

### Response Flow

```text
SSH Brute Force
      ↓
Wazuh Rule 100621
      ↓
Active Response
      ↓
firewall-drop
      ↓
iptables
      ↓
Attacker IP Blocked
      ↓
Block Removed After 60s
```

The response executes on the **victim's Wazuh agent** and blocks the attacker's IP using `iptables`.

## 🔗 Alert Timeline

The Wazuh dashboard showed the detection and response chain:

```text
SSH Authentication Failures
          ↓
Rule 5760
          ↓
Rule 100621
          ↓
Brute Force Detected
          ↓
firewall-drop
          ↓
Rule 651
          ↓
Host Blocked
```

* **5760** → SSH authentication failure
* **100621** → Custom brute-force detection
* **651** → Firewall block confirmation

## ✅ Result

The lab successfully demonstrated:

* SSH brute-force simulation
* SSH authentication log analysis
* Custom Wazuh rule creation
* Same-source-IP correlation
* MITRE ATT&CK mapping
* Automated active response
* `iptables` firewall blocking
* Alert-chain verification

The attacker IP was automatically blocked at the firewall level **without manual analyst intervention**.

## 📸 Evidence

|  # | Screenshot                                                                    | What it shows                                  |
| -: | ----------------------------------------------------------------------------- | ---------------------------------------------- |
|  1 | [Failed SSH Attempts](https://github.com/0xwasif/Wazuh-Lab/blob/main/02-attack-simulation/01-ssh-brute-force/screenshots/01-SSH-failed-attempts.png) | Repeated failed SSH authentication attempts    |
|  2 | [Ping Blocked](https://github.com/0xwasif/Wazuh-Lab/blob/main/02-attack-simulation/01-ssh-brute-force/screenshots/02-ping-blocked.png)           | Attacker traffic blocked after active response |
|  3 | [iptables Block](https://github.com/0xwasif/Wazuh-Lab/blob/main/02-attack-simulation/01-ssh-brute-force/screenshots/03-iptable-table.png)       | `iptables` DROP rule blocking the attacker IP  |
|  4 | [Alert Chain](https://github.com/0xwasif/Wazuh-Lab/blob/main/02-attack-simulation/01-ssh-brute-force/screenshots/04-Wazuh-rule-chain.png)             | Wazuh alert chain `5760 → 100621 → 651`        |
