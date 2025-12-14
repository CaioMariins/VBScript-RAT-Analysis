# 🔬 Detailed Technical Analysis: [RAT/sorvepotel]

* **sha256**: f6e94fa9e9316191168dd36a698a02f4dfd914b5533f4161f5e1377470e034c2

* **File name**: Orcamento.zip

* **First seen**: 2025-10-13 22:21:34 UTC

This repository serves as a portfolio to present an in-depth analysis of a VBScript-based malware, focusing on its multi-layered obfuscation, robust persistence, and Command and Control (C2) capabilities. The objective of this analysis is to dissect the threat's *modus operandi*, from initial execution to C2 server communication and the full roster of supported commands.

This malware has been used in recent campaigns in Brazil, sent via WhatsApp with various names (in this sample: orcamento.vbs). The campaign appears to include other file formats such as .exe and .msi and has been referred to as "sorvete no pote" (ice cream in a jar) or "sorvepotel".

---

### 1. Initial Deobfuscation (Layer 1)

The malware uses an initial VBScript obfuscation layer to hide the main *payload*.


| Original Action | Deobfuscation Method | Result |
| :--- | :--- | :--- |
| The VBScript executes `ExecuteGlobal varTJtFM` | The `ExecuteGlobal` instruction was replaced by `WScript.Echo` | The second layer code was fully revealed, allowing static analysis of the complete *payload*. |

---

### 2. Mode of Operation, Installation, and Persistence

The *malware* possesses a self-installation function that establishes triple persistence on the victim's operating system.

![Installation](img/install.png)

#### ⚙️ System Installation Details

* **Service Installation Location:** `C:\ProgramData\WindowsManager\`
* **Service File:** `WinManagers.vbs`
* **Log File:** `C:\Temp\client_log.txt`
* **Registry File:** `HKCU\Software\WinManager\Notified`

#### 🔗 Mechanisms of Persistence

The malware employs multiple techniques to ensure continuous execution:

1.  **Registry Key (Run Key):**
    * **Path:** `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\Windows Manager Services`
    * **Value:** `wscript.exe strInstallFile REG_SZ` (Execution at every user *login*).
    
![registry](img/install_reg.png)

2.  **Service:**
    * **Created Service Name:** `Windows Manager Services`
    * **Action:** The service starts immediately after installation and uses a common name to deceive the user. *"Windows Manager Services"* is a swapped name of *"Windows Services Manager"*, a legitimate Windows management service, executed via the `services.msc` binary.
    

3.  **Scheduled Task (Schtasks):**
    * **Command:** `schtasks /create /tn "Windows Manager Services" /sc onstart /ru SYSTEM /f`
    * **Detail:** Creates a scheduled task that executes the *script* at system startup, using `SYSTEM` privileges for maximum resilience. 
    
![service](img/task.png)

#### 🔒 Prevention of Multiple Executions (MUTEX)

The *malware* uses WMI to verify the execution of *scripting* processes (`'wscript.exe'` or `'cscript.exe'`), functioning as a rudimentary MUTEX mechanism to avoid redundancy.

![mutex](img/mutex.png)
---

### 3. Command and Control (C2) Communication

The C2 is the central point for data exfiltration and command issuance. This malware features a C2 address modification system to attempt IP hiding, while facilitating persistence on the victim's system.

#### 📡 C2 Discovery and Address

* **Discovery Mechanism:** The malware searches for the C2 server URL inside a **specific email**.

![info_email](img/info_email.png)

* **Temporary File:** `C:\Temp\email_out.txt` is created to store the request result containing the C2 link.

* **URL Reported in Analysis (Anyrun):** `https://shopeeship[.]com/api[.]php` (due to C2 address rotation, it cannot be guaranteed that this URL will be used regularly).

* **Detail:** The domain is registered via **Domains By Proxy**, indicating an attempt to obfuscate the threat operators.

#### 🔄 C2 Operation Cycle

After victim registration, a communication loop begins:

* **Heartbeats:** Sending life signals every 6 iterations of the main loop.
* **C2 Verification:** The email is periodically checked to validate or update the C2 server address.
* **Backup Connection:** Checks for the existence of a *failover* (backup connection) if the primary C2 does not respond. 

#### 📦 Data Exfiltration

* **Frequency and Method:** Victim data is sent in batches of **15 KB**, at **30-second** intervals, via HTTP **POST** requests to the C2.
* **Public IP:** The victim's public IP address is obtained via `api.ipify.org`.

---

### 4. Collected Victim Information

The *malware* collects a unique ID and various system *fingerprinting* information:

| Category | Collected Details |
| :--- | :--- |
| **Unique Identification** | ID generated from the HD serial number and MAC address. |
| **Network** | IP Address. |
| **System and User** | Computer Name, Username. |
| **Operating System** | Windows Version and Installation Date. |

---

### 5. Commands Supported by the C2

The presence of these commands in the code demonstrates the high level of control the C2 operator can exercise over the infected machine:

| Category | Supported Commands |
| :--- | :--- |
| **Data Collection** | `EnviarInfo`, `CaptureScreen`, `GetTaskList` |
| **File Control** | `UploadFileInChunks`, `ListFiles`, `DownloadFileFromClient`, `UploadFileToClient`, `DeleteFileORFolder`, `RenameFileOrFolder`, `RenameFileOrFolderMoveFileOrFolder`, `GetFileInfo`, `SearchFiles`, `CreatFolder` |
| **Remote Execution** | `ExecutarCMD`, `ExecutarPS` |
| **System Control** | `ReiniciarPC` (Restart PC), `DesligarPC` (Shutdown PC), `KillProcess` |
| **Malware Maintenance** | Contains a function to **update** the *malware*, enhancing the threat.|

---

## 💡 Conclusion and Security Implications

This analysis confirms that VBScript remains a viable language for developing *malware* with high evasion and persistence capabilities. The combination of simple obfuscation, triple persistence (Run Key, Service, and Scheduled Task), and a set of file control and remote execution commands demonstrates that this *malware* acts as a full-fledged Remote Access Tool (**RAT**).

**Defense Implications:** Early identification of API calls or *strings* related to registry key manipulation, service creation, and `schtasks` usage are critical security points. Detailed knowledge of the C2 communication cycle (heartbeats, email verification) is essential for creating effective network traffic detection rules. Attention must be paid to the rapid changing of C2 servers in case of infection of multiple machines in the same environment. This work aims to contribute to improving security posture by providing concrete data on the tactics, techniques, and procedures (TTPs) of this threat.