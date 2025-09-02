# Cybersecurity Homelab - End-to-End Attack & Detection (Story Walkthrough)

This repository documents a three-part homelab that simulates a real attacker-vs-defender scenario. Each step is explained in plain language, with a screenshot immediately below as evidence.

- **Attacker:** Kali Linux VM (recon + payload delivery)
- **Victim:** Windows 10 VM (instrumented with Sysmon + Splunk)
- **Outcome:** Reverse-shell activity executed and **fully detected** via Sysmon logs in Splunk, including parent/child process relationships.

> ⚠️ **Safety & Ethics:** This lab is for defensive education in an isolated environment. Offensive specifics are summarized at a high level. Do **not** run any of this outside your own isolated lab.



## Table of Contents
- [Objective](#objective)
- [System & Network Model](#system--network-model)
- [Part 1 — Provisioning & Base Network](#part-1--provisioning--base-network)
- [Part 2 — Services, Logging & Visibility](#part-2--services-logging--visibility)
- [Part 3 — Exploitation, Detection & Response](#part-3--exploitation-detection--response)
- [Detection Evidence (Splunk)](#detection-evidence-splunk)
- [Timeline Reconstruction](#timeline-reconstruction)
- [Lessons Learned](#lessons-learned)



## Objective

We built a safe, isolated lab to simulate a realistic compromise: a Kali attacker delivers a reverse-TCP payload to a Windows 10 target, while Sysmon and Splunk capture the forensic trail. This demonstrates both **offensive execution** (to generate telemetry) and **defensive detection engineering** (to detect, reconstruct, and explain what happened).



## System & Network Model

Two VirtualBox VMs live on an **Internal Network** (no internet). Windows 10 plays the endpoint we want to protect and observe; Kali plays the attacker on the same isolated segment.



## Part 1 — Provisioning & Base Network

We provisioned **two VMs** — Windows 10 (target) and Kali Linux (attacker).  
![Two VMs installed in VirtualBox]
<img width="1919" height="1079" alt="2 vm installed " src="https://github.com/user-attachments/assets/6c88f30f-c61e-4199-9c02-0ed19330dc9a" />


We configured **Windows 10** to use **Internal Network**, ensuring it only talks to other lab VMs (not the internet or host LAN).  
![Windows VM set to Internal Network]
<img width="1919" height="1079" alt="2  assigning internal network  win10" src="https://github.com/user-attachments/assets/a9d7b8c3-e91a-4f89-9b2e-0ea279927b4e" />


We set **Kali** to the **same Internal Network** so both VMs share an isolated segment for safe testing.  
![Kali VM set to Internal Network]
<img width="1919" height="1079" alt="3  assigning internal network kali" src="https://github.com/user-attachments/assets/56e67c56-a51b-47e4-a25f-85184b655783" />


We assigned a **static IP** to Windows (e.g., `192.168.20.10/24`) so scans and callbacks are predictable.  
![Windows internal network IP assigned]
<img width="1919" height="1079" alt="4  win 10 internal network IP ADDRESS" src="https://github.com/user-attachments/assets/fc8182a8-8290-4582-807f-7a72b43591bb" />


We assigned a **static IP** to Kali (e.g., `192.168.20.11/24`) so the handler/server bind to a known interface.  
![Kali internal network IP assigned]
<img width="1919" height="1079" alt="5  kali internal network IP ADDRESS" src="https://github.com/user-attachments/assets/d074536e-9bd4-490a-8656-96018bbeeb75" />


We tested connectivity **Windows → Kali** with ICMP. Successful replies confirm the path is live.  
![Ping Kali from Windows]
<img width="1919" height="1079" alt="6  ping kali" src="https://github.com/user-attachments/assets/1dd73319-cfcd-4b65-a4a4-26f5c4ead74b" />


We tested connectivity **Kali → Windows** with ICMP. Bidirectional round-trip proves the segment is working for recon and command-and-control.  
![Ping Windows from Kali]
<img width="1919" height="1079" alt="7  ping windows" src="https://github.com/user-attachments/assets/848dc4f0-6618-4118-92f6-e2a021826ee7" />




## Part 2 — Services, Logging & Visibility

On **Windows 10**, we installed **Splunk Enterprise** to index endpoint telemetry locally for analysis.  
![Splunk Enterprise installed on Windows]
<img width="1919" height="1079" alt="8  have splunk on windows 10" src="https://github.com/user-attachments/assets/a8740b0d-7d57-41d9-a62a-56cbd215f5c8" />


We installed **Sysmon** (Sysinternals) to record rich host telemetry (process creation, parent/child chains, command lines).  
![Sysmon installation step 1] <img width="1919" height="1079" alt="9  installing symson" src="https://github.com/user-attachments/assets/e0e2e1b7-cbe3-446c-8901-f9eaa1403d99" />

![Sysmon installation step 2] <img width="1919" height="1079" alt="10   installing sysmon" src="https://github.com/user-attachments/assets/0766cf8a-6eec-47e0-a804-15da46486934" />


We configured **Splunk inputs** to ingest the **Sysmon Operational** channel into a dedicated index (`endpoint`).  
![Splunk inputs.conf configured for Sysmon ingestion] <img width="1919" height="1079" alt="34  making sure splunk ahs input conf to injest data" src="https://github.com/user-attachments/assets/ea9e694c-d3fd-41c9-93ca-eb941178d358" />


We created the **`endpoint` index** in Splunk so events have a destination matching the config.  
![Created Splunk index: endpoint] <img width="1919" height="1079" alt="35  creating a new index named endpoint as per input conf" src="https://github.com/user-attachments/assets/cbe6caba-0c08-44a1-8f50-0b212ef1245a" />


We **restarted Splunkd** so the new input/index configuration takes effect and begins ingesting.  
![Restarting Splunkd service] <img width="1919" height="1079" alt="35  giving a restart to splunk d services" src="https://github.com/user-attachments/assets/d9a29d7a-7963-4ba6-8431-b5c8f0707355" />


We verified that **events are flowing** by running a basic search in Search & Reporting.  
![Data is being ingested into Splunk] <img width="1919" height="1079" alt="36  can see data being injested" src="https://github.com/user-attachments/assets/f42f00c0-78e4-4e76-877b-6843b14043dd" />


We previewed a **process GUID** so later we can correlate the malicious parent process (payload) to its spawned children (cmd, net, ipconfig).  
![Process GUID view for correlation] <img width="1919" height="1079" alt="37  seeing it through process guid" src="https://github.com/user-attachments/assets/8eac89ae-da42-4095-921e-f320c1c85390" />


We **refined searches** for readability (selecting `_time`, `ParentImage`, `Image`, `CommandLine`) so the attack chain is crystal clear.  
![Refined Splunk search for better visibility] <img width="424" height="48" alt="38  altering the search for a better view" src="https://github.com/user-attachments/assets/0764445b-7f13-4d89-8a8a-f9a3ebf15f26" />




## Part 3 — Exploitation, Detection & Response

### Reconnaissance (Attacker POV)

On **Kali**, we first **confirmed the attacker IP** so callbacks and handler bind to the correct interface.  
![Kali ifconfig showing attacker IP] <img width="1919" height="1079" alt="11  ifconfig at kali" src="https://github.com/user-attachments/assets/79381d52-fbc9-498c-b8a3-7b266f177b8b" />


We cross-checked with `ip a` to confirm interface and addressing details.  
![Kali ip a confirming interface/IP] <img width="1919" height="1131" alt="12  ip a at kali" src="https://github.com/user-attachments/assets/30cb6867-05f6-49cf-bece-ba3e4acbad76" />


We used **Nmap** to scan the Windows target and identify exposed services.  
![Initial Nmap scan against the Windows host] <img width="1914" height="1079" alt="13  nmap at kali" src="https://github.com/user-attachments/assets/cf9052b3-c6a5-41b7-b192-d770678e26b9" />


We ran a more detailed **Nmap -A** scan; we observed **RDP (3389)** among the open ports — confirming the host is up and reachable.  
![Nmap -A scan of the Windows target] <img width="1919" height="1079" alt="14  nmap -A targetting windows ip" src="https://github.com/user-attachments/assets/e52b0dc9-3d5e-4ca0-b8b2-b99f77aa6416" />




### Payload Generation (Attacker POV)

We launched the payload generator and selected a **Windows 64-bit reverse TCP** option to create a controlled reverse-shell binary for telemetry generation.  
![Launching payload generation tool] <img width="1919" height="1079" alt="15  msfvenom" src="https://github.com/user-attachments/assets/4cc97227-5595-4d0d-9a78-645e7dd32a1c" />


We reviewed **available payloads** to confirm the correct reverse-shell type for this demo.  
![Listing payload options] <img width="1919" height="1079" alt="16  msf venom payloads" src="https://github.com/user-attachments/assets/ca64690c-e4ba-4c6d-bad3-295f7eb8c025" />


We selected the **reverse TCP** choice appropriate for our Windows target.  
![Choosing reverse TCP payload] <img width="1919" height="1079" alt="17  using reverse tcp payload" src="https://github.com/user-attachments/assets/6acd4ea6-fbcf-4ceb-97d9-338239a589f6" />


We generated the binary with the attacker’s IP/port and gave it a **double-extension name** (e.g., `Resume.pdf.exe`) to study how such files appear to users and in logs.  
![Preparing the generation command (details redacted for safety)] <img width="1919" height="1079" alt="18  command to generate reverse tcp payload" src="https://github.com/user-attachments/assets/0e6d5da4-1579-4dd8-b837-80cfa07f40d3" />

We verified that the **payload file exists** and is ready to be served to the Windows VM.  
![Payload file successfully created] <img width="1919" height="1079" alt="19  malware file created" src="https://github.com/user-attachments/assets/6e35aada-3cf7-4ba8-819c-adff0dcb3815" />


> Note: This is a **defensive** exercise to generate telemetry. Offensive specifics are summarized, not provided for real-world misuse.



### Handler Setup (Attacker POV)

We opened the handler console and configured it to match our reverse-shell parameters (attacker IP/port, Windows 64-bit).  
![Setting up handler] <img width="1919" height="1079" alt="20  msf console handler for listening in the port that we have configured" src="https://github.com/user-attachments/assets/a5517c1e-cd41-4434-938f-1c6e0657a8ae" />

We switched from a generic option to the **Windows x64 reverse TCP** choice so the handler expects the right session type.  
![Changing from generic to desired payload] <img width="1919" height="1079" alt="21  changing the metsploit generic tcp " src="https://github.com/user-attachments/assets/efd4795b-c14a-4cc4-8bab-e03c5f2ff132" />

We verified the **payload option is correct** before listening.  
![Payload option confirmed] <img width="1919" height="1076" alt="22  option has changed to desired tcp" src="https://github.com/user-attachments/assets/52e04909-97b2-408f-bf61-040d11ac9175" />


We set **LHOST** to Kali’s internal IP and confirmed all options.  
![Setting LHOST for handler] <img width="1919" height="1079" alt="23  changing host ip address for handler" src="https://github.com/user-attachments/assets/912c40bf-a254-43ce-ae65-192734bf5f10" />


We started the **listener** and waited for the victim to execute the binary.  
![Exploit/handler waiting for connection] <img width="1919" height="1078" alt="24  exploit - waiting for test machine to execute our malware" src="https://github.com/user-attachments/assets/f11e594b-7259-4d3e-9c4f-7b0246ad6fa6" />




### Delivery & Execution (Victim POV)

We spun up a **Python HTTP server** on Kali in the directory containing the binary so the Windows VM can download it from the isolated network.  
![Python HTTP server serving the binary] <img width="1919" height="1078" alt="25  server for test machine to download the malware" src="https://github.com/user-attachments/assets/e50eb0e3-ec4f-4a5c-a379-54761a41f423" />


On **Windows**, we temporarily disabled **real-time protection** (lab only) so the demo could proceed and produce logs for study.  
![Turning off Windows Defender real-time protection] <img width="1919" height="1079" alt="26  turning off real time security on windows test machine" src="https://github.com/user-attachments/assets/19f070ca-4843-4f6b-9943-7eb710106241" />


From the Windows VM, we **downloaded the file** from Kali’s HTTP server.  
![Downloading the file from Kali] <img width="1919" height="1079" alt="27  test machine downloading the malware file" src="https://github.com/user-attachments/assets/6c82ea23-c1e6-406f-ba0a-06ee51a2c12c" />


We highlighted how the file can **masquerade** using a double extension (`.pdf.exe`) and why **showing file extensions** matters.  
![File shows true .exe extension after download] <img width="1919" height="1079" alt="28  after downloading you can see the file extension too its not actually  pdf its exe" src="https://github.com/user-attachments/assets/019508d1-5df7-4806-b29a-775bd5c01458" />


**Windows SmartScreen** attempted to block execution; we bypassed it (lab only) to capture telemetry for this study.  
![SmartScreen warning shown] <img width="1915" height="1078" alt="29  windows trying to protect my pc lol" src="https://github.com/user-attachments/assets/8a9c5447-80b4-4331-a5b1-b7cfc9f8e631" />

We **executed the binary** and confirmed it ran on the target.  
![Binary running successfully on Windows] <img width="1919" height="1079" alt="30  malware running successfully" src="https://github.com/user-attachments/assets/f8e329fd-59e5-48a2-8fb5-e8c1b4afe1e2" />




### Post-Exploitation (Both POVs)

Back on the attacker console, the **handler received a connection** — the reverse shell is live in our lab.  
![Handler shows session established] <img width="1919" height="1079" alt="31  we now a connection on our handler " src="https://github.com/user-attachments/assets/e34ef709-0dd3-45d8-8068-02603dcc96d1" />

We dropped to a shell and ran **basic enumeration** (`net user`) to list local accounts.  
![`net user` executed on the target] <img width="1919" height="1079" alt="32  executing 3 commands" src="https://github.com/user-attachments/assets/77c9bbe2-d3e0-42ab-aea1-58da0b82c258" />


We ran **group inspection** (`net localgroup`) to see memberships (e.g., Administrators).  
![`net localgroup` output on the target] <img width="1919" height="1079" alt="32 2" src="https://github.com/user-attachments/assets/ae7742b1-0ac6-45d2-80c0-15f16e023eec" />


We ran **`ipconfig`** to capture network configuration for situational awareness.  
![`ipconfig` output on the target] <img width="1919" height="1079" alt="33 3" src="https://github.com/user-attachments/assets/8dbad5a9-1930-494b-90a3-1a5fae113f87" />


> Every action above generates **Sysmon event code 1** (process creation) with parent/child relationships and command lines. Those events were ingested by Splunk.



## Detection Evidence (Splunk)

We confirmed **Splunk inputs** were configured for **Sysmon Operational** and pointed to the **`endpoint`** index.  
![Splunk inputs.conf for Sysmon] <img width="1919" height="1079" alt="34  making sure splunk ahs input conf to injest data" src="https://github.com/user-attachments/assets/a0026e31-4b4e-46cd-bebc-773f33b6f2fc" />


We verified the **index exists**, matching the input configuration.  
![Index 'endpoint' created] <img width="1919" height="1079" alt="35  creating a new index named endpoint as per input conf" src="https://github.com/user-attachments/assets/3085025e-f4d1-4347-8a10-dfbd6efd2080" />


We saw **events flowing** into Splunk after the service restart.  
![Sysmon events are being ingested] <img width="1919" height="1079" alt="36  can see data being injested" src="https://github.com/user-attachments/assets/567cd4be-0ada-4413-955b-bb2b11cad4ea" />


We used **Process GUID** correlation to tie the **parent process** (the downloaded executable) to **child processes** (`cmd.exe`, `net.exe`, `ipconfig.exe`).  
![Process GUID correlation in Splunk] <img width="1919" height="1079" alt="37  seeing it through process guid" src="https://github.com/user-attachments/assets/b8004bde-7755-4771-b9e9-339127075626" />


We refined the query to show a concise table of **_time**, **ParentImage**, **Image**, **CommandLine**, making the malicious chain unmistakable.  
![Refined Splunk search for chain visibility] <img width="424" height="48" alt="38  altering the search for a better view" src="https://github.com/user-attachments/assets/755416ab-be41-4c80-8794-b99b966f5173" />


Finally, Splunk shows the **full attack chain**: downloaded executable → `cmd.exe` → `net.exe` / `ipconfig.exe`.  
![Final detection chain proving parent/child process lineage] <img width="976" height="215" alt="39  now it shows everything what was done when " src="https://github.com/user-attachments/assets/4a1e5a10-ef03-4f5d-82b1-2c43ea9122ff" />


> Example Splunk query (defender-friendly):  
> ```
> index=endpoint ("*.pdf.exe" OR "cmd.exe" OR "net.exe" OR "ipconfig.exe")
> | table _time, ParentImage, Image, CommandLine
> | sort _time
> ```
> Tune for your environment (hashes, paths, users, SIGMA mappings, etc.).



## Timeline Reconstruction

- Recon: Attacker identifies its IP and scans the Windows host (Nmap).  
- Payload Prep:** A Windows-compatible reverse-shell binary is generated for telemetry testing.  
- Delivery: Binary is hosted on Kali’s HTTP server and downloaded by Windows.  
- Execution: User runs `.pdf.exe`; SmartScreen warning is bypassed (lab only).  
- C2: Handler on Kali receives a session; basic reconnaissance commands are executed.  
- Detection: Sysmon + Splunk capture and correlate the parent/child chain and command lines.



## Lessons Learned

- Isolation is everything: Internal-only networking keeps experiments safe.  
- User deception is trivial: Double extensions (`.pdf.exe`) still trick users. Enable “show file extensions.”  
- Least privilege matters: Admin users make attacker life easy; standardize non-admin daily use.  
- Application control helps: Use AppLocker/allow-listing to block unknown binaries (especially in user-write paths).  
- EDR + Logging: Keep Defender/EDR enabled with tamper protection; Sysmon + SIEM provide the forensic truth.  
- Detections to keep: Alert on suspicious parent/child combos (e.g., `*.pdf.exe` → `cmd.exe`, `cmd.exe` → `net.exe`) and unusual outbound connections from user processes.
