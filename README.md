# Verifying-the-Agent-in-the-Wazuh-Dashboard
How to verify that your Domain Controller agents are reporting correctly in the Wazuh dashboard, and how to silently upgrade them in the future without touching each server.

📊 Verifying the Agent in the Wazuh Dashboard
After the agent has been installed and the server rebooted, log into your Wazuh dashboard (e.g., https://your-wazuh-manager-ip).

1. Confirm the Agent is Connected
Open the Wazuh menu (☰) and go to Wazuh → Agents.

You’ll see a list of all agents. Find your Domain Controller by hostname.

Check the Status column – it should say Active (green dot).

If it shows Disconnected, wait a minute or two and refresh. If it stays disconnected, check network connectivity and the manager’s address in ossec.conf.

Click the agent’s name to open its details.

2. Review Agent Details
In the agent overview panel, you can quickly verify:

Version: matches the MSI you deployed.

Operating system: correct Windows Server version.

Registration IP and Manager IP are as expected.

Last keep alive: should be a few seconds ago.

3. Check Incoming Events (Security Logs)
To confirm that the agent is actually collecting the security logs you configured (via agent.conf or ossec.conf):

While viewing the agent, scroll down to the Security events section (if available) or go to Wazuh → Security events and filter by agent.name : "dc01.yourdomain.local".

Look for the exact Event IDs you enabled, e.g.:

4624 – Successful logon

4625 – Failed logon

4720 – User account created

5145 – File share access

Click on an event to see the full JSON payload – you should see rich Windows event fields like TargetUserName, LogonType, ShareName, etc.

4. Validate that the Central Configuration is Applied
If you used a group (e.g., DomainControllers) on the Wazuh manager to push the agent.conf, verify it’s active:

Go to Wazuh → Management → Groups.

Click on the group you assigned to the DCs (e.g., DomainControllers).

Inside the group, go to the Files tab and open agent.conf. It should contain the <localfile> blocks you defined.

Back on the Agents page, select your Domain Controller and look at the Configuration tab (or go to Management → Configuration, select the agent). You should see the merged configuration. Confirm that the <localfile> queries are present.

💡 Quick Tip: If you don’t see the expected events, double‑check that the GPO audit policies are still enabled. Sometimes local security policy can override the GPO (run auditpol /get /category:* on the DC to verify).

🔄 Silently Upgrading Wazuh Agents
When a new agent version is released, you have several ways to roll it out without logging into each Domain Controller. Here are the two most practical GPO‑based methods.

Method 1 – Use the GPO Software Installation “Upgrade” Feature (Recommended)
This method uses the same GPO that installed the agent. It will automatically upgrade the old agent the next time the server reboots.

Step‑by‑step:

Download the new agent MSI (e.g., wazuh-agent-4.9.0-1.msi) and place it in a network share (like your existing \\YourServer\WazuhDeploy$\NewVersion\).

Make sure Domain Computers have Read access.

Open Group Policy Management, locate your Deploy Wazuh Agent – DCs GPO, and Edit it.

Go to Computer Configuration → Policies → Software Settings → Software installation.

In the right pane, right‑click the existing Wazuh Agent package → Properties.

Go to the Upgrades tab.

Click Add and then Browse, enter the UNC path to the new MSI file, e.g.:
\\YourServer\WazuhDeploy$\NewVersion\wazuh-agent-4.9.0-1.msi

Choose Package can upgrade the existing package (or both options) and click OK.

Click OK again to close the Properties.

What happens now?
At the next system boot, Windows will automatically uninstall the old agent and install the new one. The agent service will restart and reconnect to the manager using its existing client.keys (authentication is preserved because the keys are in the Wazuh installation directory, which survives the upgrade).

To trigger the upgrade sooner, you can remotely reboot the Domain Controller during a maintenance window.

Method 2 – Use a Startup Script to Uninstall & Re‑install
If you prefer a script‑based approach or need to pass custom command‑line parameters again, use a startup script.

On your file share, create a batch file (e.g., UpgradeWazuh.bat):

batch
@echo off
:: Stop the service if running
net stop WazuhSvc /y

:: Uninstall the old agent silently
msiexec /x {PUT-YOUR-OLD-PRODUCT-CODE} /qn /norestart

:: Install the new agent silently (with manager address)
msiexec /i "\\YourServer\WazuhDeploy$\NewVersion\wazuh-agent-4.9.0-1.msi" /qn WAZUH_MANAGER="192.168.1.100" WAZUH_REGISTRATION_SERVER="192.168.1.100" /norestart

:: (Optional) copy a custom ossec.conf if needed
xcopy /Y "\\YourServer\WazuhDeploy$\ossec.conf" "C:\Program Files (x86)\ossec-agent\ossec.conf*"

net start WazuhSvc
To find your old product code, run wmic product where "name like 'Wazuh%%'" get IdentifyingNumber on a DC with the old agent installed, or check the registry under HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall.

Edit the GPO, go to Computer Configuration → Windows Settings → Scripts (Startup/Shutdown).

Replace or add this new script to the Startup list. Remove any older install script to avoid conflict.

Force a GP update (gpupdate /force) and reboot the DCs.

Method 3 – Remote Upgrade from the Wazuh Manager (No GPO Needed)
If your agents are online and your manager is version 4.x, you can use the Wazuh API to upgrade agents remotely. This is extremely convenient for large environments.

On the Wazuh manager CLI, run:

bash
/var/ossec/bin/agent_upgrade -a <agent_id> -f /path/to/wazuh-agent-4.9.0-1.msi
Replace <agent_id> with the ID of the Domain Controller (e.g., 001).
You must first upload the MSI installer to the manager (usually to /var/ossec/var/upgrade/).

For bulk upgrades, use the Wazuh dashboard:

Go to Wazuh → Management → Agents.

Select multiple agents, click the Upgrade button, choose the package, and confirm.

The upgrade will be scheduled and can be performed with a live restart. Note that for Windows agents, the Wazuh manager needs SMB connectivity to a share containing the MSI, or you can use the “online repository” method if your agents have internet access.

⚠️ Important: The remote upgrade method (without GPO) requires the agent to be Active and the manager to have a way to distribute the MSI (HTTP, local file, or SMB). For closed environments, GPO‑based upgrade is often simpler and more reliable.

🔁 Post‑Upgrade Verification
After any upgrade:

Wait a few minutes for the agents to reconnect.

In the Wazuh dashboard, check the Agents list – the Version column should now reflect the new number.

Click the agent and confirm the Last keep alive is recent.

Spot‑check a few security events to ensure log collection is still working.
