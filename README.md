# Microsoft Defender for Endpoint: Enterprise Endpoint Security

**Deutsche Kurzbeschreibung:** Deployment und Konfiguration von Microsoft Defender for Endpoint in einer Lab-Umgebung. Umfasst Device Onboarding, Sicherheitsrichtlinien (Next-Generation Protection, Firewall), Detection Testing mit PowerShell-Alerts und Incident Investigation. Dokumentiert einen vollständigen EDR-Workflow von der Geräteanmeldung bis zur Alert-Analyse im Defender-Portal.

---

## What This Project Is About

Endpoint security is where most breaches actually happen. An unmanaged or misconfigured endpoint is a door left open, regardless of how strong your perimeter defenses are. This project was about proving I understand how to deploy, configure, and monitor endpoint protection using Microsoft Defender for Endpoint in an enterprise environment.

The goal wasn't just to onboard a device and call it done. It was to build a complete detection and response workflow: enroll the device, configure security policies, test detection capabilities with a known malicious command, investigate the alert in the portal, and verify that protection features like real-time scanning and cloud protection are actively enforcing security. That cycle, from onboarding to alert investigation, is what makes endpoint security relevant.

One note upfront: this project uses Microsoft Defender for Business, which is the SMB-focused variant included in Microsoft 365 Business Premium. It's deliberately simplified compared to the full Defender for Endpoint Plan 2 that enterprises deploy. Features like Attack Surface Reduction rules, advanced hunting, and custom detection rules aren't available in the portal. The core endpoint protection, policy enforcement, and alert investigation capabilities are all present, which is what this project demonstrates.

---

## Architecture

![Architecture Diagram](https://github.com/lionelmsango/defender-for-endpoint-lab/blob/e76a9d434907afaaad57c272544eb9b8ff64e8ca/screenshots/mde_architecture_svg.svg)

```
Microsoft 365 Tenant (LioDefenderLab.onmicrosoft.com)
│
├── Microsoft Defender for Business
│   ├── Device Onboarding: Local Script (PowerShell)
│   ├── Next-Generation Protection: Real-time protection, Cloud protection
│   ├── Firewall: Windows Defender Firewall enabled
│   └── Incident & Alert Management
│
├── Enrolled Endpoint
│   └── desktop-k3qoano (VMware · Windows 11 Pro 25H2)
│       ├── Microsoft Defender Antivirus: Active
│       ├── Real-time protection: ON
│       └── Cloud-delivered protection: ON
│
└── Detection Test
    └── PowerShell command line alert (ID 1: Execution incident on one endpoint)
```

---

## Phase 1 — Tenant Setup and Device Onboarding

Before any policies or detection tests could happen, the tenant needed to exist and the device needed to be enrolled. The old tenant from previous projects (LioHomeLab.onmicrosoft.com) expired, so I created a fresh Microsoft 365 Business Premium trial tenant: LioDefenderLab.onmicrosoft.com.

The Defender portal (security.microsoft.com) was immediately accessible with the admin account. The first step was confirming the onboarding wizard was available under Endpoints → Device onboarding. Microsoft's setup wizard can only be run once per tenant, so getting this right the first time mattered.

**Screenshot 1 — Defender portal homepage showing SOC optimization dashboard:**

![Defender Portal Homepage](screenshots/01_defender_portal_homepage_png.jpg)

The onboarding method selection offers four options: Intune (for MDM-enrolled devices), Local Script (PowerShell), Group Policy, and VDI scripts. For a single-device lab, Local Script is the straightest path. Group Policy would require an Active Directory domain, which adds unnecessary complexity for a one-VM setup.

**Screenshot 2 — Onboarding method selection (Local Script chosen):**

![Onboarding Method Selection](screenshots/02_onboarding_method_selection_png.jpg)

After selecting Local Script, the portal generated a downloadable package: `WindowsDefenderATPLocalOnboardingScript.cmd`. I copied this to the VM (MDE-LAB-VM at this point, later renamed to desktop-k3qoano), ran it as Administrator in PowerShell, and watched the onboarding process execute.

The script output showed:
```
Successfully onboarded machine to Microsoft Defender for Endpoint
```

**Screenshot 3 — Onboarding script executed successfully on the VM:**

![Onboarding Success](screenshots/03_onboarding_success_png.jpg)

Within 5–10 minutes, the device appeared in the Defender portal's Device Inventory under Assets → Devices. The device name showed as `desktop-k3qoano` (the VM had been renamed after initial setup), status Active, risk level Medium.

**Screenshot 4 — Device inventory showing enrolled device (desktop-k3qoano):**

![Device Inventory](screenshots/04_device_inventory_png.jpg)

The Medium risk level is expected for a freshly onboarded device with no alert history. Risk levels adjust based on active alerts and discovered vulnerabilities over time.

---

## Phase 2 — Detection Testing with EICAR-Style PowerShell Alert

Onboarding proves the device is talking to the portal. Detection testing proves the security features are actually working. Microsoft provides a safe test command that triggers an alert without executing real malware. It's the endpoint security equivalent of an EICAR test file.

I created a test folder (`C:\test-MDATP-test`) and ran the official detection test command from Microsoft's documentation:

```powershell
powershell.exe -NoExit -ExecutionPolicy Bypass -WindowStyle Hidden $ErrorActionPreference = 'silentlycontinue';(New-Object System.Net.WebClient).DownloadFile('http://127.0.0.1/1.exe', 'C:\test-MDATP-test\invoice.exe');Start-Process 'C:\test-MDATP-test\invoice.exe'
```

This command attempts to download a fake file from localhost and execute it. It's designed to trigger multiple detection signatures: suspicious PowerShell execution, obfuscated command line, and process creation from a script-downloaded binary.

Within seconds, an alert appeared in the Incidents queue.

**Screenshot 5 — Alert detected: "Suspicious PowerShell command line":**

![Test Alert Detected](screenshots/05_test_alert_detected_png.jpg)

The alert was categorized under Investigation & Response → Incidents, with severity Medium, category Execution, and status Active. The incident was automatically assigned ID 1 and labeled "Execution incident on one endpoint."

**Screenshot 6 — Alert details showing incident graph and impacted assets:**

![Alert Details Investigation](screenshots/06_alert_details_investigation_png.jpg)

The Attack Story tab shows two alerts linked to this incident:
1. "[Test Alert] Suspicious Powershell commandline"
2. "Suspicious PowerShell command line"

The incident graph visualizes the attack chain: the initial PowerShell process (127.0.0.1 connection) linked to the impacted device (desktop-k3qoano). The technique profile on the right identifies this as "Malicious use of PowerShell" with two impacted assets: the device and the user account (lione).

This detection workflow, from command execution to alert generation to incident investigation, is exactly what happens in a real environment when Defender detects suspicious activity. The difference is that in production, the PowerShell command would be an actual attack, not a test.

---

## Phase 3 — Policy Configuration: Next-Generation Protection and Firewall

Detection capabilities are only half of endpoint security. The other half is preventive policy enforcement. Microsoft Defender for Business organizes endpoint protection into two main policy categories: Next-Generation Protection (antivirus, real-time scanning, cloud protection) and Firewall (network-level access control).

### Next-Generation Protection (NGP)

The NGP policy (`NGP Windows default policy`) was already configured by default when the tenant was created. I verified that the critical protection features were enabled:

- **Real-time protection:** ON — scans files and processes as they execute
- **Block at first sight:** ON — submits unknown files to Microsoft's cloud protection service for instant analysis

These two settings are the foundation of modern endpoint protection. Real-time protection stops known threats immediately. Block at first sight catches zero-day threats by leveraging Microsoft's global threat intelligence.

**Screenshot 7 — NGP policy showing real-time protection and cloud protection enabled:**

![NGP Policy Configuration](screenshots/07_ngp_policy_configuration_png.jpg)

One limitation worth noting: Defender for Business doesn't expose Attack Surface Reduction (ASR) rules in the portal. ASR rules block specific risky behaviors like Office macros, script execution from email, and credential theft attempts. They're a standard feature in Defender for Endpoint Plan 2, but in Defender for Business, you'd need to configure them through Intune or Group Policy if you wanted granular control.

### Firewall Policy

The Firewall policy (`Firewall Windows default policy`) was also pre-configured and enabled. The default configuration turns on Windows Defender Firewall for domain, private, and public networks, which is correct for most environments.

**Screenshot 8 — Firewall policy status showing enabled state:**

![Firewall Policy Status](screenshots/08_firewall_policy_status_png.jpg)

The firewall policy and NGP policy are both visible under Endpoints → Configuration management → Device configuration. This is the central location for viewing all policies applied to enrolled devices.

---

## Phase 4 — Device Risk Assessment and Active Alerts

After onboarding, detection testing, and policy verification, the next step was examining the device's overall security posture from the portal's perspective. The Device Overview page aggregates all security-relevant information about an endpoint into one view.

**Screenshot 9 — Device risk overview showing Medium risk level with active alerts:**

![Device Risk Overview](screenshots/09_device_risk_overview_png.jpg)

The device (desktop-k3qoano) shows:
- **Risk level:** Medium
- **Active alerts (Last 180 days):** 2 active alerts, 1 active incident
- **Exposure level:** Info (0 active security recommendations)

The Medium risk level is driven by the two active alerts from the PowerShell detection test. This is the expected behavior: Defender assigns risk scores based on unresolved security events, even if those events are test alerts. In a production environment, you'd investigate and close these incidents, which would lower the risk level back to Low.

The Exposure level of "Info" indicates no critical vulnerabilities have been detected yet. This makes sense for a freshly built VM with no installed software beyond the OS and Defender itself.

**Screenshot 10 — Device Incidents and Alerts tab showing execution incident:**

![Device Alerts Tab](screenshots/10_device_alerts_tab_png.jpg)

The Incidents and Alerts tab lists the same execution incident visible in the main Incidents queue. This view is device-specific, which is useful when investigating a single endpoint's alert history. In an environment with hundreds of devices, you'd use this tab to answer "what has this specific machine been doing?"

---

## Phase 5 — Vulnerability Management and Secure Score

Two additional Defender dashboards provide long-term security visibility: Vulnerability Management and Secure Score. Neither had meaningful data yet because the tenant was only a few hours old, but documenting what these dashboards measure is part of understanding the full platform.

### Vulnerability Management

The Vulnerability Management dashboard (Exposure management → Vulnerability management → Overview) tracks known CVEs affecting enrolled devices. It scans software inventory, compares installed versions against Microsoft's vulnerability database, and produces remediation recommendations.

**Screenshot 11 — Vulnerability management dashboard (0/100 exposure score, no data yet):**

![Vulnerability Management Dashboard](screenshots/11_vulnerability_management_dashboard_png.jpg)

The Endpoint exposure score shows **0/100**, which is the default state for a new tenant. Vulnerability scanning runs daily, so it takes 24–48 hours before discovered vulnerabilities and recommendations start populating this dashboard. In a production environment with hundreds of devices, this dashboard would show high-priority CVEs that need patching, such as outdated browser versions, unpatched OS builds, or vulnerable third-party software.

### Secure Score

Microsoft Secure Score (Exposure management → Secure Score → Overview) measures the tenant's overall security configuration across identity, devices, apps, and data. It's a percentage score based on how many of Microsoft's recommended security controls are enabled.

**Screenshot 12 — Secure Score dashboard showing 0% (calculating, tenant too new):**

![Secure Score Dashboard](screenshots/12_secure_score_dashboard_png.jpg)

The score shows **0%** with a message: "Microsoft is calculating your Secure Score, which usually takes 2-4 days when your tenant was created." This is expected behavior. Secure Score calculation requires baseline telemetry from enrolled devices, which takes a few days to accumulate after tenant creation.

Once populated, Secure Score provides actionable recommendations like "Enable MFA for all users," "Turn on tamper protection," or "Configure retention policies for Exchange." Each recommendation includes the point value it would add to the score, which lets admins prioritize high-impact changes.

---

## Project Summary

| Component | Status |
|-----------|--------|
| Tenant setup (LioDefenderLab.onmicrosoft.com) | Done |
| Device onboarding (Local Script method) | Done |
| Detection test (PowerShell alert generated) | Done |
| Alert investigation (incident graph reviewed) | Done |
| NGP policy verification (real-time + cloud protection ON) | Done |
| Firewall policy verification (enabled) | Done |
| Device risk assessment (Medium risk, 2 active alerts) | Done |
| Vulnerability management dashboard accessed | Done |
| Secure Score dashboard accessed | Done |

**Detection workflow validated:** Suspicious command executed → Alert generated (Suspicious PowerShell command line) → Incident created (ID 1: Execution incident on one endpoint) → Investigation performed in portal → Impacted assets identified (device + user).

---

## Lessons Learned

**Windows 11 25H2 requires manual MDE Sense Client installation.** Microsoft removed the Defender for Endpoint Sense Client from the base Windows 11 image starting with 24H2. If you try to onboard a 25H2 or later VM without first running `DISM /online /Add-Capability /CapabilityName:Microsoft.Windows.Sense.Client~~~~`, the onboarding script will execute without error but the device won't appear in the portal. This isn't documented in the onboarding wizard itself, which cost me about an hour of troubleshooting before finding the Microsoft support article.

**Defender for Business is not Defender for Endpoint Plan 2.** This limitation became apparent when searching for Attack Surface Reduction rules, custom detection rules, and advanced hunting features. They simply don't exist in the Defender for Business portal. The product is intentionally simplified for SMB environments. For a portfolio project targeting enterprise IT admin roles, this means the project demonstrates endpoint protection and alert investigation, but not the advanced threat hunting capabilities that larger organisations rely on. Worth knowing before claiming "full MDE experience" on a resume.

**Portal navigation doesn't match most online documentation.** Microsoft's official documentation assumes you're using Defender for Endpoint Plan 2, which has a different portal structure. Features are in different menus, some aren't available at all, and some documentation paths just don't exist in Defender for Business. For example, "Vulnerability management" isn't a top-level menu option; it's nested under Exposure management → Vulnerability management → Overview. If you're following a tutorial and can't find a setting, check whether the guide assumes Plan 2 or Business.

**Detection tests need to be run from the enrolled device.** This seems obvious in hindsight, but the Microsoft documentation doesn't emphasize it. If you run the PowerShell detection command from your host machine instead of the enrolled VM, nothing happens. The alert only triggers when the command executes on an endpoint that's actively reporting to Defender. Simple mistake, easy to miss if you're managing multiple VMs.

**Secure Score and Vulnerability Management take 2–4 days to populate.** Both dashboards showed "no data" immediately after device onboarding, which is expected but not well-documented. Vulnerability scanning runs on a daily schedule, and Secure Score calculation requires baseline telemetry. If you're building this lab and wonder why your dashboards are empty, wait 48 hours. It's not broken; it's calculating.

---

## Next Steps — Expanding This Project

**Attack Surface Reduction (ASR) rules via Intune.** Defender for Business doesn't expose ASR rules in the portal, but they can be configured through Intune device configuration policies. ASR rules block specific attack vectors like Office macro execution, script-based attacks, and credential dumping. Integrating this with the Intune project would create a complete endpoint hardening workflow.

**Automated investigation and response testing.** Defender for Endpoint includes automated investigation (AIR) features that analyze alerts, correlate events, and execute remediation actions without human intervention. Testing this would require generating multiple related alerts to trigger the automated investigation engine.

**Threat and vulnerability management remediation.** After the vulnerability scan populates (48 hours post-onboarding), the next step would be addressing the discovered vulnerabilities. This might include patching outdated software, removing deprecated applications, or updating the OS build. Documenting a full vulnerability remediation cycle would demonstrate proactive security management, not just alert reaction.

**Integration with Microsoft Sentinel.** Defender for Endpoint can forward alerts and telemetry to Microsoft Sentinel (Microsoft's cloud-native SIEM). Connecting these two would enable advanced hunting queries, custom analytics rules, and cross-platform threat correlation. This would be particularly relevant combined with the Wazuh SIEM project, creating a multi-platform security monitoring environment.

**Mobile device onboarding (iOS/Android).** Defender for Endpoint supports mobile device enrollment for both company-owned and BYOD scenarios. Testing mobile onboarding would require either a physical device or an Android emulator, and would demonstrate endpoint protection extending beyond traditional desktops.

---

## Related Projects

| Project | Description |
|---------|-------------|
| [Microsoft Intune: Modern Workplace Management](https://github.com/lionelmsango/intune-modern-workplace-lab) | Device compliance and conditional access — the foundation this project's endpoint protection builds on |
| [Microsoft 365 Administration & IAM](https://github.com/lionelmsango/m365-administration-lab) | Tenant setup, MFA, and identity protection baseline |
| [Wazuh SIEM – Security Monitoring](https://github.com/lionelmsango/wazuh-siem-lab) | SIEM layer that could integrate with Defender alert data for cross-platform threat detection |
| [pfSense Firewall Administration](https://github.com/lionelmsango/pfsense-firewall-lab) | Network-layer security complementing endpoint-layer protection |
