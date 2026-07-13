# Crowie-Honeypot-Wazuh-SIEM
This project demonstrates the deployment and integration of an SSH honeypot using Cowrie with Wazuh SIEM for centralized security monitoring, log collection, and attacker behavior analysis.
The goal of this laboratory environment is to simulate an exposed SSH service, collect malicious activity, and analyze attacker techniques through a SIEM platform.

Architecture
                 Internet
                    |
                    |
              SSH Attacks
                    |
                    v
          +----------------+
          |    Cowrie      |
          | SSH Honeypot   |
          +----------------+
                    |
              cowrie.json
                    |
                    v
          +----------------+
          |     Wazuh      |
          |  Logcollector  |
          +----------------+
                    |
                    v
          +----------------+
          | Wazuh Indexer  |
          +----------------+
                    |
                    v
          +----------------+
          | Wazuh Dashboard|
          +----------------+
Project Objectives
Deploy an SSH honeypot environment
Collect attacker interaction logs
Integrate Cowrie JSON events into Wazuh SIEM
Analyze authentication attacks
Build visibility into malicious activity
Practice SOC monitoring workflows
Technologies Used
Component	Purpose
Cowrie	SSH/Telnet honeypot
Wazuh SIEM	Security monitoring and detection
Wazuh Indexer	Event storage
Wazuh Dashboard	Visualization and investigation
Linux	Host operating system
JSON Logging	Event format
Environment

Operating System:

Linux Server

Security Components:

Cowrie SSH Honeypot
Wazuh Manager
Wazuh Indexer
Wazuh Dashboard

Log source:

/home/cowrie/my-honeypot/var/log/cowrie/cowrie.json

Log format:

JSON
Wazuh Configuration

The Wazuh manager was configured to collect only Cowrie events.

Configured log source:

<localfile>
  <log_format>json</log_format>
  <location>/home/cowrie/my-honeypot/var/log/cowrie/cowrie.json</location>
</localfile>

The following modules were disabled to create a dedicated honeypot monitoring sensor:

Syscollector
Rootcheck
File Integrity Monitoring
Vulnerability Detection

Reason:

The objective of this deployment is focused exclusively on attacker activity collected by Cowrie.

Security Events Collected

The honeypot captures:

Failed SSH authentication attempts
Username enumeration
Password guessing activity
Attacker commands
Session information
Source IP addresses
Malware download attempts

Example events:

cowrie.login.failed
cowrie.command.input
cowrie.session.connect
Detection Capabilities

The environment can be used to identify:

SSH Brute Force

Indicators:

Multiple failed login attempts
Repeated username/password combinations
Same source IP targeting SSH
Attacker Behavior Analysis

Collected information:

Commands executed
Tools downloaded
Attack patterns
Session duration
Troubleshooting Examples
Wazuh cannot read Cowrie logs

Problem:

Permission denied

Cause:

The Wazuh service user did not have access to the Cowrie directory.

Solution:

Configured ACL permissions:

setfacl -m u:wazuh:rx /home/cowrie
Indexer connection issues

Problem:

IndexerConnector initialization failed

Cause:

Missing or incorrect Indexer authentication configuration.

Solution:

Verified:

Wazuh Indexer availability
Credentials
TLS certificates
Keystore configuration
Skills Demonstrated
SIEM deployment
Linux administration
Log collection and parsing
Security monitoring
Honeypot deployment
Incident investigation
Threat detection
Blue Team operations
Future Improvements

Possible improvements:

Add custom Wazuh detection rules
Map Cowrie events to MITRE ATT&CK techniques
Automate IP reputation checks
Create threat intelligence enrichment
Build automated attack reports
Author

Cybersecurity Laboratory Project

Focus areas:

SOC Analysis
Defensive Security
Threat Detection
SIEM Engineering
