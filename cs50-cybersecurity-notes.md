# CS50 Cybersecurity — Course-Only Notes
*David J. Malan · Related slide headings grouped for a compact, three-part refresher.*

Only examples shown in the course slides are included. Page divisions are suggested; printed length depends on formatting.

## Page 1 — Securing Accounts & Securing Data

### Securing Accounts

**Authentication • Authorization • Usernames • Passwords**  
Authentication checks who you are; authorization controls what you’re allowed to do. A username identifies your account; a password helps prove it’s yours.

**Dictionary Attacks • Brute-Force Attacks**  
Dictionary attacks try likely passwords. Brute force tries every combination. Longer passwords increase the work—but predictable passwords are still easy targets.  
**From the slides:** four digits give **10,000** possibilities. Four characters chosen from 94 possibilities give **78,074,896**. These counts assume every combination is allowed; they don’t make predictable choices safe.

**National Institute of Standards and Technology (NIST)**  
The guidance quoted in the course favors long passwords, blocking common or compromised choices, limiting failed attempts, and avoiding arbitrary password changes. Security questions and publicly accessible password hints are weak spots. Treat the quoted requirements as course-era guidance.

**Two-Factor Authentication (2FA) • Multi-Factor Authentication • Knowledge • Possession • Inherence**  
Use different kinds of proof: something you know, something you have, or something you are. More than one factor means a stolen password alone may not be enough.

**One-Time Password (OTP) • SIM Swapping**  
OTPs are codes used once. SMS-based authentication depends on control of your phone number. SIM swapping lets an attacker take over that number and potentially receive your codes.

**Keylogging • Credential Stuffing**  
Keylogging records what you type. Credential stuffing tries leaked credentials on other services. Reusing passwords lets one breach spread into several account takeovers.

**Social Engineering • Phishing • Machine-in-the-Middle Attacks**  
Attackers don’t always need to break the technology. They can trick you into sharing information or place themselves between you and a legitimate service.

**Single Sign-On (SSO) • Password Managers**  
SSO lets one identity handle multiple services. Password managers help you use unique passwords without memorizing them all. Both make the account protecting that access especially important.

**Passkeys • WebAuthn**  
Passkeys use public/private keys rather than a shared password. With WebAuthn, the service sends a challenge, your authenticator signs it, and the service verifies the signature. The private key isn’t sent to the website.

### Securing Data

**Passwords • Hashing • One-Way Hash Functions • Cryptographic Hash Functions**  
Hashing turns input into a fixed-length value. It isn’t reversible encryption. Password storage should use a suitable salted, costly password-hashing method so stolen hashes are harder to attack.

**Dictionary Attacks • Brute-Force Attacks • Rainbow Tables • Salting**  
Attackers can hash guesses and compare them with stolen hashes. Rainbow tables use precomputed work. A unique salt prevents the same precomputed work from applying neatly across accounts.  
**From the slides:** Carol and Charlie both use `cherry`. Without salts, their hashes match. With different salts, their stored hashes differ—even though the passwords are identical.

**Cryptography • Codes • Encode/Decode • Ciphers • Encipher/Encrypt • Decipher/Decrypt • Keys**  
Codes substitute meanings; ciphers transform messages. Encryption converts plaintext into ciphertext, and decryption reverses it using the appropriate key. Encoding alone isn’t a security guarantee.

**Secret-Key Cryptography • Secret-Key Encryption • Symmetric-Key Encryption • Decrypting • Cryptanalysis**  
Symmetric encryption uses a shared secret key. Cryptanalysis studies how to break the protection.  
**From the slides:** shifting letters by 13 turns `BE SURE TO DRINK YOUR OVALTINE` into `OR FHER GB QEVAX LBHE BINYGVAR`. It demonstrates a cipher—not a secure modern encryption method.

**Public-Key Cryptography • Public-Key Encryption • Asymmetric-Key Encryption • RSA**  
Public-key encryption uses a key pair: encrypt with the recipient’s public key, decrypt with their private key. RSA is one algorithm discussed in the course.

**Key Exchange • Diffie-Hellman**  
Key exchange allows parties to establish a shared secret over a network without simply transmitting that secret. Diffie-Hellman demonstrates how both sides can calculate the same secret from private values and exchanged public values.

**Digital Signatures • Sign • Verify**  
A private key creates a signature; the public key verifies it. Signatures help establish authenticity and detect modification. They don’t hide the message.

**Encryption in Transit • End-to-End Encryption**  
Transit encryption protects data while it travels. End-to-end encryption keeps the content protected between the communicating endpoints, rather than making the intermediary a place where it’s readable.

**Deletion • Secure Deletion • Full-Disk Encryption • Encryption at Rest**  
Deleting a file doesn’t necessarily erase its underlying data. Secure deletion aims to prevent recovery. Full-disk encryption protects stored information by making it unreadable without the required key.

**Ransomware • Quantum Computing**  
Ransomware uses encryption against the victim to deny access to data. Quantum computing raises questions about the future security of some existing cryptographic methods—not whether every form of encryption suddenly becomes useless.

---

## Page 2 — Securing Systems

### Connections and trust

**Encryption • Wi-Fi • Wi-Fi Protected Access**  
Wi-Fi protection secures the wireless connection. It isn’t a substitute for protecting communication all the way to the service you’re using.

**HTTP • Machine-in-the-Middle Attacks • Packet Sniffing**  
Unencrypted HTTP can expose traffic to observation and modification. Packet sniffing captures traffic; an attacker in the middle may also change it.  
**From the slides:** an HTTP checkout request contains `number=4242424242424242`. The point is that sensitive information in an unencrypted request can be visible on the network.

**Cookies • Session Hijacking**  
Cookies can carry the identifier that keeps you logged in. If someone gets a usable session identifier, they may impersonate your session without entering your password.  
**From the slides:** the server sends `Set-Cookie: session=1234abcd`; the browser later sends `Cookie: session=1234abcd`. That exchange shows why protecting the session token matters.

**HTTPS • TLS**  
HTTPS uses TLS to protect communication and authenticate the website’s domain. It protects the connection; it doesn’t establish that everything on the website is trustworthy.

**Certificate • X.509 • Certificate Authority (CA)**  
A certificate connects an identity to a public key. X.509 defines a common certificate format. Certificate authorities and signature verification help browsers decide which certificates to trust.

**SSL Stripping • HSTS**  
SSL stripping tries to keep communication on insecure HTTP. HSTS tells the browser to insist on HTTPS. The slides also show `includeSubDomains` and `preload`, which extend that protection.

**VPN • SSH**  
A VPN creates a protected tunnel to another network endpoint. SSH provides encrypted remote access to a computer.  
**From the slides:** Malan connects using `ssh stanford.edu` and runs `date` remotely. The demonstration shows that commands are now running on the other computer.

### Network access

**IP Address • Port • Port Scanning**  
An IP address identifies a network endpoint. Ports distinguish services. Port scanning checks which services appear reachable. The slides highlight **22, 80, and 443**—commonly associated with SSH, HTTP, and HTTPS.

**Penetration Testing • Ethical Hacking**  
These use security-testing techniques with authorization. Permission and scope distinguish legitimate testing from unauthorized intrusion.

**Firewall • Deep Packet Inspection**  
A firewall controls which traffic is allowed. Deep packet inspection examines more of the traffic’s contents rather than relying only on basic addressing information.

**Proxy**  
A proxy sits between parties and forwards requests. That position can support filtering or monitoring, but it also makes the intermediary part of the trust picture.

### Malicious software and disruption

**Malware • Virus • Worm**  
Malware is malicious software. A virus infects other files or programs; a worm can propagate between systems on its own.

**Botnet**  
A botnet is a collection of compromised devices controlled together. Attackers can use the combined resources rather than relying on a single machine.

**Denial-of-Service Attack (DoS) • Distributed Denial-of-Service Attack (DDoS)**  
These attacks aim to make a service unavailable. A distributed attack uses multiple sources, making it harder to stop by blocking one sender.

**Antivirus • Automatic Updates • Zero-Day Attacks**  
Antivirus can detect threats, but it can’t guarantee protection. Updates close known vulnerabilities. Zero-day attacks exploit weaknesses before an effective fix is available to defenders.

**Main lesson:** no single layer covers everything. Encryption, access controls, updates, and malware defenses solve different parts of the problem.

---

## Page 3 — Securing Software & Preserving Privacy

### Securing Software

**Phishing**  
Visible link text doesn’t prove where a link goes.  
**From the slides:** a link displays `https://harvard.edu` but its actual destination is `https://yale.edu`. Check the destination, not just the label.

**Code Injection • Cross-Site Scripting (XSS) • Reflected • Stored**  
Injection happens when supplied data becomes executable instructions. Reflected XSS comes back through a response; stored XSS persists and reaches later visitors.  
**From the slides:** a search-results message normally includes `cats`. Replacing that input with script content demonstrates what happens when the page interprets input as code instead of text.

**Character Escapes • Content-Security-Policy**  
Escaping special characters can keep content as text in the appropriate context. CSP restricts permitted sources for scripts and other resources.  
**From the slides:** `&lt;` represents `<`, so script-like text can be displayed rather than interpreted as an HTML tag.

**SQL Injection • Prepared Statements**  
Building SQL by inserting raw input lets that input alter the query. Prepared statements with bound parameters keep input separate from SQL instructions.  
**From the slides:** a query built around `username = '{username}'` is replaced with one using `username = ?` and a separately supplied value.

**Command Injection • `system` • `eval`**  
These functions can turn strings into commands or code. Passing untrusted input into them can let an attacker control what executes.

**Developer Tools • Client-Side Validation • Server-Side Validation**  
Browser restrictions aren’t trustworthy enforcement. Validate input and permissions on the server.  
**From the slides:** removing `disabled` from a checkbox or `required` from a text input bypasses those browser-side restrictions.

**Cross-Site Request Forgery (CSRF) • GET • POST**  
CSRF tricks a logged-in browser into sending an unwanted request. Changing GET to POST isn’t enough by itself.  
**From the slides:** an automatically submitted form illustrates the problem; a `csrf_token` illustrates a defense.

**Open Worldwide Application Security Project (OWASP)**  
OWASP provides resources for understanding common application-security risks and defenses.

**Arbitrary Code Execution (ACE) • Remote Code Execution (RCE) • Buffer Overflow • Stack Overflow**  
Memory errors can go beyond crashing an application: they may let an attacker redirect execution. RCE means execution can be triggered remotely.  
**From the slides:** stack diagrams show an overwritten return address redirecting execution toward attacker-controlled code.

**Cracking • Reverse Engineering • Malware Analysis**  
Cracking bypasses protections. Reverse engineering examines how software works. Malware analysis applies investigation techniques to malicious software.

**Open-Source Software • Closed-Source Software**  
Source availability affects who can inspect software; it doesn’t automatically establish that one model is secure and the other isn’t.

**App Stores • Package Managers • Operating Systems • Bug Bounty**  
Software distribution, signature verification, operating-system protections, and vulnerability-reporting programs all contribute to security.

**CVE • CVSS • EPSS • KEV**  
CVE identifies a vulnerability. CVSS describes severity. EPSS estimates exploitation likelihood. KEV lists vulnerabilities known to be exploited. They provide different information—not interchangeable scores.

### Preserving Privacy

**Web Browsing History • Logs • HTTP Headers • User-Agent**  
Browsers and servers leave records. Headers can reveal browser information and where a request came from.  
**From the slides:** `Referer` can reveal a referring page; referrer policies limit what gets sent.

**Fingerprinting**  
A combination of browser and device characteristics can help identify a visitor, even without a conventional tracking cookie.

**Session Cookies • Tracking Cookies • Tracking Parameters • Third-Party Cookies • Supercookies**  
Cookies may support logins or tracking. URL parameters can identify clicks. Third-party resources can connect visits across different sites; supercookies use more persistent tracking mechanisms.  
**From the slides:** Harvard, Yale, and Stanford pages load an image from the same third-party domain. Requests to that domain carry its cookie, allowing it to connect visits across those pages.

**Private Browsing**  
Private browsing mainly limits information retained locally after the session. It doesn’t make activity invisible to websites or network operators.

**DNS • DNS over HTTPS (DoH) • DNS over TLS (DoT)**  
DNS translates names into addresses. DoH and DoT encrypt communication with the DNS resolver; they don’t make all internet activity anonymous.

**Virtual Private Network (VPN) • Tor**  
VPNs move traffic through an intermediary. Tor uses multiple relays to improve privacy. Both change who can observe parts of the communication; neither removes every privacy risk.

**Permissions • Location-Based Services**  
Check what apps can access, especially location. Granting permission also grants an opportunity to collect information about you.

*Sources: official course slides for [Accounts](https://cdn.cs50.net/cybersecurity/2023/x/lectures/0/lecture0.pdf), [Data](https://cdn.cs50.net/cybersecurity/2023/x/lectures/1/lecture1.pdf), [Systems](https://cdn.cs50.net/cybersecurity/2023/x/lectures/2/lecture2.pdf), [Software](https://cdn.cs50.net/cybersecurity/2023/x/lectures/3/lecture3.pdf), and [Privacy](https://cdn.cs50.net/cybersecurity/2023/x/lectures/4/lecture4.pdf). Notes are paraphrased; examples are drawn from the slides.*
