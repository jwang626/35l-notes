#W10L2
# Introduction

Security is the programmer's responsibility (CS136).

We need to know 2 things:
1. **Security Model** (what you're defending) or white hat
2. **Threat Model** (who's attacking and how) or black hat

## How is "security" defined?

1. **Confidentiality/ Privacy** (don't want info leaking out)
2. **Integrity** (don't want tampering/info coming in)
3. **Availability** (is the system even up?)

## Security Model

What we need to identify:
- Assets (valuable info we're trying to protect)
- Vulnerabilities (in accessing assets)
- Threats (ways attackers can exploit vulnerabilities to access assets)

# Threat Modeling 

## Intro

 Main classifications of threats:
- insiders
- social engineering
- hardware device attacks (ex. USBs with viruses, keyboard sniffers)
- network attacks (this is what we will talk about )

Network attacks include phishing, denial of service (DoS), buffer overruns, cross-site scripting, prototype pollution

## Most Common Security Vulnerabilities

According to OWASP:
- **Broken Access Control**: access unauthorized functionality or data
	- direct-object references like altering URLs, parameters
	- inadequate session management
- **Crypto Failures**: failing to protect sensitive data
	- sending data over HTTP not HTTPS
	- using weak crypto algorithms
	- not validating TLS/SSL certificates 
		- *certificates*: digital passports for websites, facilitate encrypted connections between web servers and web browsers (employed by HTTPS)
		- when a web browser connects to a website encrypted/secured with TLS, it checks the validity of the website's certificate
- **Injection**: attacker sends malicious data as part of a command or query
	- SQL injection
	- need to sanitize/validate data!
- **Security Misconfigurations**: can happen at any layer of app stack
	- maintenance ports open, leaving default account/passwords
- **Outdated & Vulnerable Components**
- **ID & Authentication failures** 
	- easy passwords, public usernames, allowing password guessing
- **Software & Data Integrity Failures**: tampered data/programs 
	- You should be doing integrity checks via crypto certifications. also called supply chain certification
- **Security Logging & Monitoring Failures**
- **Server-Side Request Forgery (SSRF)**: attacker abuses the functionality of a server, causing it to make unintended requests to internal or external resources 
	- often combined w injection
	- trick the server to send requests to other backend servers (ex Database Server) that it wouldn't normally send. response gets sent back to the client browser
	- since server is intended to be firewall between public world and proprietary world, SSRF breaks into proprietary world
	
## Mitigation Techniques

- Input Validation and Sanitization
- Granting Least Privileges
- Regular Patching and Updates
- Network Segmentation and Firewalling


Now we'll talk more about each network attack.

# More on Network Attacks
## Cross-Site Scripting (XSS)

This is different from SSRF (this is browser-side)
Client talks to origin server, which tells client (via some malicious JS script) to send request to different server. This different server could be under different management. (attacker injects malicious script into user's browser to redirect to other site.)
- maybe one server is trying to attack other server

### Same-Origin Policy 

A potential fix which says browser can only send requests to same origin server (doesn't allow cross-site scripting)
- this imposes huge limitations, don't allow servers to collaborate

<ins>Terminology:</ins>
**Origin**: combination of the protocol (such as HTTP or HTTPS), the domain, and the port number of a web page.
Clients send **origin requests** (HTTP requests) to servers. this includes the HTTP method (GET, POST..), URL, headers, body.

### Cross-Origin Resource Sharing (CORS)

Protocol to allow collaboration. When browser attempts a cross-origin request, it notifies the different server
-talks directly between servers? less latency?

## Prototype Pollution

[FCC Javascript Prototypes](https://www.freecodecamp.org/news/a-beginners-guide-to-javascripts-prototype/)

We need to delve into JS internals for this.
- every JS object has a property `[[Prototype]]` that references the obj prototype 
	- purpose: to provide inheritance
	- Prototypal Instantiation
```js
function Animal(){
  let animal = Object.create(Animal.prototype);
  ...
  return animal
}
Animal.prototype.eat = function(){...}
```
- at the top of the prototype chain is the **root prototype**: `Object.prototype` from which all JS objects inherit unless explicitly changed
- When you write `const obj = {}` you are cloning the root
- attacks could use argument `"__proto__"`
- Mitigation: can freeze prototypes with `Object.freeze(Object.prototype)` at the start

# Testing Security

This is different from testing functionality (since bad guys are focusing on bugs). Static analysis is a big deal.

Testing needs to look for **breaking abstraction boundaries**, which is internal details of a component or module are exposed and directly manipulated by other parts of the system (ex. prototype pollution).

Lets go through the abstraction layers for this simple example: 
Read password into buffer (char array), checks, keep going if OK.
- issues: need to clear buffer when done 
	- clear buffer with `memset(buf, 0, size_of buf) `
- but clearing buffer not enough, contents are in registers too
	- clear registers `explicit_memset(same params)`, will not be optimized away

Example continued:
Lets say we have the password buffer sitting in 2 virtual memory pages. Then we can realign the buffer so that we only have to guess one character at a time via string comparison.

## Important Note on Timing Attacks/Encryption

When encrypting, it is very important that algorithms are **constant time**. That way, attackers can't figure out info based on time it takes for a system or application to respond to various inputs.
- Mitigation: introduce noise

## Spectre/ Meltdown Attack

These target vulnerabilities in CPU's cache architecture (& how it responds to invalid mem accesses) to get sensitive info.
- more severe than targeting pages, since cache miss takes much less cycles than page fault
- time how quickly it takes to figure out memory contents (TIMING is the basis of this attack)
- defense: lie about what time it is
- mitigation: separate kernel memory and user processes

# Risk Assessment

## Overview

There are too many types of attacks to take care of all, need to strategize.

Common problems 
- lack of human training
- requirements (neglecting, not negotiating w user security issues)
- ARCHITECTURE: no threat model 
- DESIGN: no design awareness of security 
- CODING: prototype pollution, SQL injection, repudiation, etc
- TESTING: no static/dynamic checking/no fuzz testing/no penetration analysis
- DEPLOYMENT: insecure default configuration/logout doesn't work properly/wrong ports enabled
- MAINTENANCE: forget to repeat the above

Checklists are your friend!

## Trusted Computing Base (TCB)

We have a large system, application inside system, that is not all that trustworthy. 
Inside the app, we have a small core part that is a trusted computing base. This is isolated from the rest of the system.
- includes OS kernel, hardware, firmware, other security-critical software

*What do you put in the TCB?*
- one side: bigger --> more functionality
- but trade-off: smaller --> more security

Questions:
- who built the stuff in TCB (Red Hat on seasnet)
- what tools did they use? (gcc, make, ...)





