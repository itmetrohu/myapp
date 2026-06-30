Palo Alto Firewall Automation Guide (Standalone)
Overview
This document provides a complete step-by-step guide for automating a standalone Palo Alto firewall (vsys1) using Python and Ubuntu Linux. It covers environment setup, API authentication, object creation, policy automation, troubleshooting, and best practices.
________________________________________
1. Environment Setup (Ubuntu)
Install Python and Required Tools
sudo apt update
sudo apt install -y python3 python3-venv python3-pip
Create Project Directory
mkdir -p ~/pan-automation
cd ~/pan-automation
Create Virtual Environment
python3 -m venv .venv
source .venv/bin/activate
Using a virtual environment ensures clean dependency management and avoids conflicts with system Python.
Install Required Python Libraries
pip install -U pip
pip install requests python-dotenv
We use: - requests for HTTPS API calls - python-dotenv for secure credential management
________________________________________
2. Secure Credential Management (.env File)
Create a .env file:
nano ~/pan-automation/.env
Example configuration:
PAN_HOST=10.0.0.69
PAN_API_KEY=YOUR_API_KEY_HERE
VERIFY_TLS=false
Secure the file:
chmod 600 .env
Best Practice: - Use an API key instead of storing username/password - Use a dedicated automation account - Restrict management access to the automation server IP
________________________________________
3. Understanding the PAN-OS XML API
All automation actions require:
•	type=config (modifying configuration)
•	action=set (writing data)
•	xpath= (location in config tree)
•	element= (XML content)
•	key= (API authentication)
Important Concept: Changes are written to the candidate configuration. A commit is required to activate them.
________________________________________
4. Standalone Firewall XPath Structure (vsys1)
Standalone firewall rulebase path:
/config/devices/entry[@name='localhost.localdomain']/vsys/entry[@name='vsys1']
Common subpaths:
Object Type	XPath
Address Object	…/vsys1/address
Address Group	…/vsys1/address-group
Service Object	…/vsys1/service
Security Rules	…/vsys1/rulebase/security/rules
Panorama uses different paths (shared / device-group). This guide is for standalone only.
________________________________________
5. Base Python API Functions
Reusable API structure:
import os
import requests
from dotenv import load_dotenv

load_dotenv("/root/pan-automation/.env", override=True)

PAN_HOST = os.getenv("PAN_HOST")
PAN_API_KEY = os.getenv("PAN_API_KEY")
VERIFY_TLS = os.getenv("VERIFY_TLS", "false").lower() == "true"

def api_get(params: dict) -> str:
    url = f"https://{PAN_HOST}/api/"
    r = requests.get(url, params=params, verify=VERIFY_TLS, timeout=45)
    r.raise_for_status()
    return r.text

def config_set(xpath: str, element: str) -> str:
    return api_get({
        "type": "config",
        "action": "set",
        "xpath": xpath,
        "element": element,
        "key": PAN_API_KEY,
    })

def commit():
    return api_get({
        "type": "commit",
        "cmd": "<commit></commit>",
        "key": PAN_API_KEY,
    })
________________________________________
6. Creating Address Objects
Example:
VSYS_BASE = "/config/devices/entry[@name='localhost.localdomain']/vsys/entry[@name='vsys1']"

name = "AO_SRC_1"
ip = "10.10.10.10"

xpath = f"{VSYS_BASE}/address/entry[@name='{name}']"
element = f"<ip-netmask>{ip}</ip-netmask>"

print(config_set(xpath, element))
________________________________________
7. Creating Address Group
members = ["AO_SRC_1", "AO_SRC_2"]

def xml_members(items):
    return "".join([f"<member>{i}</member>" for i in items])

xpath = f"{VSYS_BASE}/address-group/entry[@name='AG_SRC_AUTOMATION']"
element = f"<static>{xml_members(members)}</static>"

print(config_set(xpath, element))
________________________________________
8. Creating Service Object (TCP/1030)
xpath = f"{VSYS_BASE}/service/entry[@name='TCP_1030']"
element = "<protocol><tcp><port>1030</port></tcp></protocol>"

print(config_set(xpath, element))
________________________________________
9. Creating Security Policy Rule (Combined App Block)
Blocking TikTok and Telnet in one rule:
RULE_NAME = "BLOCKED_APPS"
FROM_ZONES = ["trust"]
TO_ZONES = ["untrust"]
APPS = ["tiktok", "telnet"]

RULES_BASE = f"{VSYS_BASE}/rulebase/security/rules"

xpath = f"{RULES_BASE}/entry[@name='{RULE_NAME}']"

element = (
    f"<from>{xml_members(FROM_ZONES)}</from>"
    f"<to>{xml_members(TO_ZONES)}</to>"
    f"<source><member>any</member></source>"
    f"<destination><member>any</member></destination>"
    f"<application>{xml_members(APPS)}</application>"
    f"<service><member>application-default</member></service>"
    f"<action>deny</action>"
    f"<log-start>no</log-start>"
    f"<log-end>yes</log-end>"
)

print(config_set(xpath, element))
Important: Ensure deny rules are placed above broad allow rules.
________________________________________
10. Commit Changes
print(commit())
Commit validates the entire configuration tree.
________________________________________
11. Troubleshooting Scenarios
XPath Schema Error
Occurs when using Panorama path on standalone firewall. Fix: Use vsys1 standalone XPath.
DHCP Commit Validation Error
Example:
deviceconfig -> system -> type -> dhcp-client is missing 'accept-dhcp-hostname'
Fix via CLI:
configure
set deviceconfig system type dhcp-client accept-dhcp-hostname no
commit
API Connectivity Issues
•	Verify HTTPS reachability
•	Confirm management profile allows your IP
•	Confirm API enabled
________________________________________
12. Best Practices
•	Use dedicated automation admin account
•	Store API key securely
•	Restrict mgmt interface access
•	Validate candidate config before commit
•	Log automation output
•	Understand rule order impact
________________________________________
13. Skills Demonstrated
This automation workflow demonstrates:
•	Linux environment configuration
•	Python virtual environment management
•	Secure credential handling
•	API-driven firewall configuration
•	Address and service object automation
•	Policy rule automation
•	XPath schema understanding
•	Candidate vs running configuration knowledge
•	Troubleshooting commit validation errors
________________________________________
14. Future Enhancements
•	Insert rules at specific positions
•	Bulk object import from CSV
•	Config backup automation
•	Rule hit count analysis
•	Panorama device-group automation
•	Integration with CI/CD pipeline
________________________________________
End of Guide
