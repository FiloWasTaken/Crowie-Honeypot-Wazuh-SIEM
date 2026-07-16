# Honeypot Cowrie + Wazuh SIEM — Detection Lab

Technical documentation of an SSH/Telnet honeypot setup based on **Cowrie**, monitored through **Wazuh SIEM**, exposed on a public VPS to collect and analyze real attacks coming from the internet.

> **Disclaimer**: this project is for cybersecurity research/learning purposes (threat intelligence, detection engineering). The honeypot does not provide access to production systems and does not contain any sensitive data of its own; however, it does contain real data collected from actual attacks (source IPs, attempted credentials, commands executed by attackers), since it is exposed on a public VPS on the internet.

---

## Table of contents

- [Project goal](#project-goal)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Setup — Cowrie](#setup--cowrie)
- [Setup — Wazuh](#setup--wazuh)
- [Cowrie → Wazuh log integration](#cowrie--wazuh-log-integration)
- [Verifying it works](#verifying-it-works)
- [IP lookup and command analysis](#ip-lookup-and-command-analysis)
- [Security considerations](#security-considerations)

---

## Project goal

The goal of this project is to simulate a vulnerable SSH/Telnet service in order to:

- Observe and collect unauthorized access attempts coming from the internet (credential stuffing, brute force, post-login commands).
- Centralize and analyze the honeypot's logs through a SIEM (Wazuh), to get hands-on with a realistic **detection engineering** flow (log ingestion → parsing → alerting).
- Build a small case study to showcase in a portfolio/CV context.

---

## Architecture

Cowrie and Wazuh (Manager + Dashboard) are installed **on the same machine/VM**, a VPS publicly exposed on the internet.

```
                        INTERNET (real attackers)
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   Public VPS            │
                    │                         │
                    │  ┌───────────────────┐  │
                    │  │ Cowrie (honeypot) │  │
                    │  │ SSH / Telnet      │  │
                    │  └────────┬──────────┘  │
                    │           │ JSON log     │
                    │           ▼              │
                    │  ┌───────────────────┐  │
                    │  │ Wazuh Manager     │  │
                    │  │ (localfile)       │  │
                    │  └────────┬──────────┘  │
                    │           ▼              │
                    │  ┌───────────────────┐  │
                    │  │ Wazuh Dashboard   │  │
                    │  └───────────────────┘  │
                    └─────────────────────────┘
```
Wazuh Manager:
<br>
<img width="1354" height="567" alt="image" src="https://github.com/user-attachments/assets/9bb60f23-4fae-45cc-a1a4-278016a927cc" />

Cowrie:
<br>
<img width="569" height="84" alt="image" src="https://github.com/user-attachments/assets/b3eb4c87-7dc2-41b2-b57f-999e80cea6e9" />

---

## Tech stack

| Component | Role |
|---|---|
| **Cowrie** | Medium-interaction SSH/Telnet honeypot, simulates a fake shell and filesystem |
| **Wazuh Manager** | Ingests, parses (decoder/rule) and generates alerts from Cowrie logs |
| **Wazuh Dashboard** | Centralized visualization of alerts and logs |
| **Public VPS** | Hosts both services, exposed on the internet to collect real attacks |

Deliberately minimal stack: no additional integrations (no TheHive, no Shuffle, no external notifications) — the focus is on honeypot → SIEM.

---

## Setup — Cowrie

*(Cowrie is already installed and configured on this machine. This is a descriptive section of the existing setup, not a step-by-step to be re-run.)*

- Installed in a dedicated Python virtual environment (unprivileged user, best practice recommended by Cowrie to limit impact in case of a sandbox escape).
- Main configuration in `cowrie.cfg` (listening ports, fake hostname/system, simulated filesystem).
- Application logs generated in **JSON** format (`var/log/cowrie/cowrie.json`), in addition to the classic text log (`cowrie.log`).
- The VPS's real SSH port was moved to a non-standard port (2222), with Cowrie listening on the "decoy" port 22 to attract automated scanners and bots that scan the default port.

<img width="1740" height="64" alt="image" src="https://github.com/user-attachments/assets/da37add9-4ce7-4bd4-8fc8-ad6c9f8ac258" />

Output of `cowrie.json` showing a captured login attempt (username, password, source IP).

---

## Setup — Wazuh

Staring at raw JSON logs in a terminal is pretty rough, so we install a SIEM on top of it.

*(Wazuh is also already installed and configured. This is a descriptive section, not an installation guide.)*

- Wazuh Manager and Dashboard installed on the same VPS as the honeypot.
- Central configuration in `/var/ossec/etc/ossec.conf`.
- The local agent (self-monitoring), active by default on the manager itself, is also used to read Cowrie's logs via `localfile`.

Rules created to generate alerts for Cowrie (new logins and commands executed once inside):

<screenshot>
Wazuh Dashboard showing the agent/host online and healthy ("Active" status).
</screenshot>

---

## Cowrie → Wazuh log integration

The integration was implemented via the **`<localfile>`** directive in Wazuh's `ossec.conf`, pointed directly at the JSON log file generated by Cowrie:

<img width="588" height="98" alt="image" src="https://github.com/user-attachments/assets/a49cdeb0-33fd-4437-9f5c-f5fdf2a86d21" />

This lets Wazuh read every new line of the Cowrie log in real time and pass it into the internal parsing/decoding pipeline, without needing additional agents (Filebeat) or syslog forwarding: since Cowrie and Wazuh run on the same machine, reading the file directly is the simplest and most effective solution.

<screenshot>
Excerpt of `ossec.conf` with the configured `<localfile>` block (confirmation of the configuration actually in use).
</screenshot>

---

While setting all of this up I ran into a problem: Wazuh's log collector was reading Cowrie's logs, but I couldn't see anything on the dashboard, because there was no rule defined for the alerts — events and alerts are two very different things. So I created two rules to generate events: one for a successful login (Cowrie accepts any credentials by design), and one to see which commands were executed once the bot got into the honeypot. I say "bot" because these are usually infected computers/VPSs running automated scripts that perform brute force; once inside, they run one or two commands, and this information is then sent back to the actual attacker.

Login rule:
<br>
<img width="689" height="169" alt="image" src="https://github.com/user-attachments/assets/aa207c8b-ed7d-4143-b0bb-5be1ecad6ac7" />

Executed-command rule:
<br>
<img width="829" height="252" alt="image" src="https://github.com/user-attachments/assets/4d79dfb5-949b-4bd0-a998-5e490fbb7008" />

After leaving port 22 open for a few days, we can see that we received roughly 12,900 alerts:

<img width="1486" height="512" alt="image" src="https://github.com/user-attachments/assets/0d78662b-39ec-42ed-935c-a2fb8411f2cb" />

---

## Verifying it works

This section documents proof that the full pipeline (Cowrie → log → Wazuh → alert) works correctly end-to-end, with real attacks intercepted by the honeypot exposed on the internet.

### 1. Connection attempts intercepted by Cowrie

Log showing an incoming SSH session from a real external IP (with timestamp and source IP visible).
<br>
<img width="1465" height="532" alt="image" src="https://github.com/user-attachments/assets/f9b054b5-4941-48aa-9a03-6bf90aad2dd4" />

### 2. Commands executed by the attacker in the fake shell

Let's filter the logs for the commands executed by this IP address using this query: `rule.id: 100200 and data.src_ip: "141.11.88.243"`
where `rule.id` specifies the rule we created earlier, and the second field filters by the IP address we're interested in.

### 3. Alert generated on the Wazuh Dashboard

The timestamps might not line up exactly, since the bot tried to get into the server many times.
<img width="1482" height="475" alt="image" src="https://github.com/user-attachments/assets/4885728d-a758-42b3-9fec-4bb59dbde9d5" />

---

## IP lookup and command analysis

After collecting the events from Cowrie, I did a manual analysis of some of the more interesting IPs and commands, to better understand the nature of the attacks received.

### 1. Lookup of attacker IP addresses

Taking the IP address from the screenshots above, a lookup on AbuseIPDB shows that this address is indeed known to be malicious:
<br>
<img width="1556" height="786" alt="image" src="https://github.com/user-attachments/assets/2eb47663-981a-42c1-b7fa-5ca48a578125" />

### 2. Commands executed by attackers in the simulated shell

Analysis of the commands actually typed by attackers once "inside" Cowrie's fake shell (useful to understand intent: reconnaissance, malware download attempts, persistence attempts, etc.).
We can see that the command executed was `uname -s -v -n -r -m`, a command used to gather information about the machine that was just "breached" — most likely, this information is then sent back to the actual attacker.

### 3. Searching for a specific command executed by a specific IP

In the case above, the only command executed was `uname -s -v -n -r -m`, but what if we want to look for specific commands?
The following query lets us do that: `rule.id: 100200 and <command>`,
for example, `rule.id: 100200 and whoami` returns the following result:
<img width="1509" height="446" alt="image" src="https://github.com/user-attachments/assets/caaa2b3c-5316-41e2-bb3d-d78f0cd8269f" />

---

## Security considerations

- The honeypot is logically isolated from the rest of the system (dedicated unprivileged user, Cowrie sandbox).
- The VPS's real SSH port was moved to a non-standard port to separate it from Cowrie's "decoy" port.
- Since Wazuh runs on the same exposed machine, it represents a potential secondary attack surface: access to the Dashboard should be monitored (authentication, firewalling ports 443/55000) to prevent it from becoming an attack vector itself.
- No real data or valid credentials are present on the machine.
