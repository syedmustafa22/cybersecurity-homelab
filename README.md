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
![Windows VM set to Internal Network](./images/part1/2_assigning_internal_network_win10.png)

We set **Kali** to the **same Internal Network** so both VMs share an isolated segment for safe testing.  
![Kali VM set to Internal Network](./images/part1/3_assigning_internal_network_kali.png)

We assigned a **static IP** to Windows (e.g., `192.168.20.10/24`) so scans and callbacks are predictable.  
![Windows internal network IP assigned](./images/part1/4_win10_ip.png)

We assigned a **static IP** to Kali (e.g., `192.168.20.11/24`) so the handler/server bind to a known interface.  
![Kali internal network IP assigned](./images/part1/5_kali_ip.png)

We tested connectivity **Windows → Kali** with ICMP. Successful replies confirm the path is live.  
![Ping Kali from Windows](./images/part1/6_ping_kali.png)

We tested connectivity **Kali → Windows** with ICMP. Bidirectional round-trip proves the segment is working for recon and command-and-control.  
![Ping Windows from Kali](./images/part1/7_ping_win.png)



## Part 2 — Services, Logging & Visibility

On **Windows 10**, we installed **Splunk Enterprise** to index endpoint telemetry locally for analysis.  
![Splunk Enterprise installed on Windows](./images/part2/8_splunk_win10.png)

We installed **Sysmon** (Sysinternals) to record rich host telemetry (process creation, parent/child chains, command lines).  
![Sysmon installation step 1](./images/part2/9_installing_symson.png)  
![Sysmon installation step 2](./images/part2/10_installing_sysmon.png)

We configured **Splunk inputs** to ingest the **Sysmon Operational** channel into a dedicated index (`endpoint`).  
![Splunk inputs.conf configured for Sysmon ingestion](./images/part2/34_inputs_conf.png)

We created the **`endpoint` index** in Splunk so events have a destination matching the config.  
![Created Splunk index: endpoint](./images/part2/35_endpoint_index.png)

We **restarted Splunkd** so the new input/index configuration takes effect and begins ingesting.  
![Restarting Splunkd service](./images/part2/35_restart_splunkd.png)

We verified that **events are flowing** by running a basic search in Search & Reporting.  
![Data is being ingested into Splunk](./images/part2/36_data_ingested.png)

We previewed a **process GUID** so later we can correlate the malicious parent process (payload) to its spawned children (cmd, net, ipconfig).  
![Process GUID view for correlation](./images/part2/37_process_guid.png)

We **refined searches** for readability (selecting `_time`, `ParentImage`, `Image`, `CommandLine`) so the attack chain is crystal clear.  
![Refined Splunk search for better visibility](./images/part2/38_better_search_view.png)



## Part 3 — Exploitation, Detection & Response

### Reconnaissance (Attacker POV)

On **Kali**, we first **confirmed the attacker IP** so callbacks and handler bind to the correct interface.  
![Kali ifconfig showing attacker IP](./images/part3/11_ifconfig_at_kali.png)

We cross-checked with `ip a` to confirm interface and addressing details.  
![Kali ip a confirming interface/IP](./images/part3/12_ip_a_at_kali.png)

We used **Nmap** to scan the Windows target and identify exposed services.  
![Initial Nmap scan against the Windows host](./images/part3/13_nmap.png)

We ran a more detailed **Nmap -A** scan; we observed **RDP (3389)** among the open ports — confirming the host is up and reachable.  
![Nmap -A scan of the Windows target](./images/part3/14_nmap_A_targetting_windows_ip.png)



### Payload Generation (Attacker POV)

We launched the payload generator and selected a **Windows 64-bit reverse TCP** option to create a controlled reverse-shell binary for telemetry generation.  
![Launching payload generation tool](./images/part3/15_msfvenom.png)

We reviewed **available payloads** to confirm the correct reverse-shell type for this demo.  
![Listing payload options](./images/part3/16_msf_venom_payloads.png)

We selected the **reverse TCP** choice appropriate for our Windows target.  
![Choosing reverse TCP payload](./images/part3/17_using_reverse_tcp_payload.png)

We generated the binary with the attacker’s IP/port and gave it a **double-extension name** (e.g., `Resume.pdf.exe`) to study how such files appear to users and in logs.  
![Preparing the generation command (details redacted for safety)](./images/part3/18_command_to_generate_reverse_tcp_payload.png)

We verified that the **payload file exists** and is ready to be served to the Windows VM.  
![Payload file successfully created](./images/part3/19_malware_file_created.png)

> Note: This is a **defensive** exercise to generate telemetry. Offensive specifics are summarized, not provided for real-world misuse.



### Handler Setup (Attacker POV)

We opened the handler console and configured it to match our reverse-shell parameters (attacker IP/port, Windows 64-bit).  
![Setting up handler](./images/part3/20_msf_console_handler.png)

We switched from a generic option to the **Windows x64 reverse TCP** choice so the handler expects the right session type.  
![Changing from generic to desired payload](./images/part3/21_changing_the_metsploit_generic_tcp.png)

We verified the **payload option is correct** before listening.  
![Payload option confirmed](./images/part3/22_option_has_changed_to_desired_tcp.png)

We set **LHOST** to Kali’s internal IP and confirmed all options.  
![Setting LHOST for handler](./images/part3/23_changing_host_ip_address_for_handler.png)

We started the **listener** and waited for the victim to execute the binary.  
![Exploit/handler waiting for connection](./images/part3/24_exploit_waiting.png)



### Delivery & Execution (Victim POV)

We spun up a **Python HTTP server** on Kali in the directory containing the binary so the Windows VM can download it from the isolated network.  
![Python HTTP server serving the binary](./images/part3/25_server_for_test_machine.png)

On **Windows**, we temporarily disabled **real-time protection** (lab only) so the demo could proceed and produce logs for study.  
![Turning off Windows Defender real-time protection](./images/part3/26_turning_off_real_time_security.png)

From the Windows VM, we **downloaded the file** from Kali’s HTTP server.  
![Downloading the file from Kali](./images/part3/27_test_machine_downloading_the_malware_file.png)

We highlighted how the file can **masquerade** using a double extension (`.pdf.exe`) and why **showing file extensions** matters.  
![File shows true .exe extension after download](./images/part3/28_after_downloading_you_can_see_file_extension.png)

**Windows SmartScreen** attempted to block execution; we bypassed it (lab only) to capture telemetry for this study.  
![SmartScreen warning shown](./images/part3/29_windows_trying_to_protect_my_pc.png)

We **executed the binary** and confirmed it ran on the target.  
![Binary running successfully on Windows](./images/part3/30_malware_running_successfully.png)



### Post-Exploitation (Both POVs)

Back on the attacker console, the **handler received a connection** — the reverse shell is live in our lab.  
![Handler shows session established](./images/part3/31_we_now_have_connection_on_handler.png)

We dropped to a shell and ran **basic enumeration** (`net user`) to list local accounts.  
![`net user` executed on the target](./images/part3/32_executing_3_commands.png)

We ran **group inspection** (`net localgroup`) to see memberships (e.g., Administrators).  
![`net localgroup` output on the target](./images/part3/32.2.png)

We ran **`ipconfig`** to capture network configuration for situational awareness.  
![`ipconfig` output on the target](./images/part3/33.3.png)

> Every action above generates **Sysmon event code 1** (process creation) with parent/child relationships and command lines. Those events were ingested by Splunk.



## Detection Evidence (Splunk)

We confirmed **Splunk inputs** were configured for **Sysmon Operational** and pointed to the **`endpoint`** index.  
![Splunk inputs.conf for Sysmon](./images/part2/34_inputs_conf.png)

We verified the **index exists**, matching the input configuration.  
![Index 'endpoint' created](./images/part2/35_endpoint_index.png)

We saw **events flowing** into Splunk after the service restart.  
![Sysmon events are being ingested](./images/part2/36_data_ingested.png)

We used **Process GUID** correlation to tie the **parent process** (the downloaded executable) to **child processes** (`cmd.exe`, `net.exe`, `ipconfig.exe`).  
![Process GUID correlation in Splunk](./images/part2/37_process_guid.png)

We refined the query to show a concise table of **_time**, **ParentImage**, **Image**, **CommandLine**, making the malicious chain unmistakable.  
![Refined Splunk search for chain visibility](./images/part2/38_better_search_view.png)

Finally, Splunk shows the **full attack chain**: downloaded executable → `cmd.exe` → `net.exe` / `ipconfig.exe`.  
![Final detection chain proving parent/child process lineage](./images/part3/39_now_it_shows_everything_what_was_done.png)

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
