+++
title = 'Aqua Threat'
description = "This one I forgot to get the description. It was an IoT/SCADA setup with multiple flags. The goal was to contaminate the water supply, thus compromise the pump. The description said something about Product Family."
category = 'IoT/SCADA'
solved = false
+++

| Info        	|                  	|
|-------------	|------------------	|
| Category    	| {{< category >}}  |
| Description 	| {{< desc >}}      |

> [!CAUTION] TL;DR
> This challenge was purely recon.
> The flag was in the product page on the manufacturer's website, accessible via the serial number found on the pump.

## Initial Recon
The water pump had a stcker with a number on it.
Maybe an access code, maybe a serial number?

## Dead Ends
The access panel had a web page open to see and control the pump.

### Remote Access
One field was to enter a remote access code.
We tried many codes with saw, such as the number on the side of the pump, with no success.
The remote access was probably for another challenge further along the IoT path.

### FTP
The web app also exposed a way to get a file by opening an FTP server on your machine and giving the SCADA system your IP and user/password.
By opening an FTP server on my machine and letting it be anonymous, we got a `.pcap` file with nothing interesting on it, at least for this challenge, except for the IP of the SCADA system which we found by monitoring connections.
This IP was going to be found either way at the end of the challenge, where it was given.

## Solution

We talked to the designer at the end of the competition to find out how to solve the challenge.

1. **Note the serial number** on the side of the pump.
This was the number we found at first.

2. **Go to the manufacturer's website** and search their products for the exact pump using the serial number.

3. **Read the technical data** on the product page and find the flag.
It was in the Product Family property, as hinted by the original description of the challenge.

Yes, it was that simple, but we missed it because we were trying to find vulnerabilities or a way to gain control on the system...

## Takeaways

### Methodology

- I need to do more recon before jumping straight into trying to find a direct path to a flag.
Sniff around, find information about the target that could be useful down the line.
