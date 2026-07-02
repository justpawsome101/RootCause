---
description: Simulated phishing on myself, using my own machine and accounts.
---

# Phishing Basics with GoPhish

### What was done

Simulated a credential-harvesting phishing campaign end to end, then extended the exercise into a second stage involving payload delivery via a C2 beacon, to get practice with the full attack chain a real engagement would use.

### Infrastructure Setup

The original plan was to host everything on an Oracle Cloud free tier VPS to keep the attack infrastructure off my host machine. This didn't end up working because Oracle had no resources available for the free tier. I had already wasted so much time with Oracles nonsense so I decided to change my setup. I resorted to using my own Ubuntu host machine as the "VPS" for the lab, accepting that this is a lab only exercise (a real engagement would need proper isolated infrastructure).

### GoPhish Setup

Installed [GoPhish](https://github.com/gophish/gophish/releases/) (v0.12.1) on the Ubuntu host. On first run it generates a random admin password and self-signed certs for the admin interface, and spins up:

* Admin panel on `https://127.0.0.1:3333`
* Phishing server on `http://0.0.0.0:80`

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption><p>GoPhish running in terminal.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption><p>GoPhish Dashboard.</p></figcaption></figure>

**Tunneling with ngrok:** Since the phishing server needed to be internet reachable for targets to hit the landing page, I signed up for ngrok and used it to forward a public HTTPS URL to the local GoPhish phishing server. This gave a working public link (`https://drinkable-status-amigo.ngrok-free.dev/`) without needing to deal with port forwarding or a real domain, which made life way easier.

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption><p>ngrok running.</p></figcaption></figure>

**Sending Profile:** Configured an SMTP sending profile using Gmail's SMTP relay (`smtp.gmail.com:587`). Sent a test email to confirm delivery worked before building the rest of the campaign.

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption><p>Configured sending profile.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (22).png" alt=""><figcaption><p>Sending test email.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption><p>Test email successful.</p></figcaption></figure>

**Email Template :"Security Alert":**

* Envelope sender spoofed as `Riot Games <*****123@gmail.com>`
* Subject: _"Urgent: Account scheduled for deletion"_
* Body: a short urgency-driven message claiming the account would be deleted in 24 hours due to reports, with a "Keep my account" link pointing to `{{.URL}} (variable that points to my ngrok URL).`

This is a classic urgency + account loss scenario, chosen because it doesn't require much context about the target and reliably drives quick clicks.

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption><p>Configuring email.</p></figcaption></figure>

**Landing Page  (Cloned Riot Games Login):** Used GoPhish's "Import Site" feature to clone the Riot Games authentication page. The live Riot login is built in React, and the imported clone didn't function correctly as a static import. Worked around this by taking the imported markup and asking an AI assistant to rewrite it as static HTML that preserved the visual design and the login form fields, which then correctly captured submitted credentials into GoPhish (Capture Submitted Data and Capture Passwords both enabled). Redirect was set to the real `leagueoflegends.com` site to reduce suspicion after capture.

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption><p>Imported Riot login page.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption><p>Rendering issue due to static import of React elements.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption><p>Cloned Page.</p></figcaption></figure>

**Targets:** Set up target group in Users & Groups with three test recipients (bob, PAwsome, Sigma Hacker aka me,myself and I) to use as the send list.

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption><p>Configured targets.</p></figcaption></figure>

**Campaign Runs:**

* **Test 2:** 3 emails sent, 1 opened, 1 clicked, 1 submitted data. This confirmed the full chain worked and was tracked live on GoPhish's campaign dashboard.

<figure><img src="../.gitbook/assets/image (30).png" alt=""><figcaption><p>Configuring campaign details.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (31).png" alt=""><figcaption><p>Campaign successfully scheduled.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption><p>Email received.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (32).png" alt=""><figcaption><p>Campaign Results.</p></figcaption></figure>

### Extra section: Sliver C2

To extend the exercise beyond credential harvesting, I set up Sliver C2 to simulate a malicious download delivered from the same phishing flow.

**Setup:**

* Downloaded and ran the Sliver server as a daemon on an ubuntu server VM, generated a new operator profile, and connected a client via multiplayer mode (imported the operator config).
* Started a listener (`https -L 0.0.0.0 -l 8443` .
* Generated a Windows/amd64 HTTP beacon (`generate beacon --http ... --os windows --arch amd64 --format exe --save /tmp/beacon.exe`) with symbol obfuscation enabled.
* Installed ngrok on this box as well to expose the Sliver listener publicly, giving a second forwarding URL for beacon callback.

<figure><img src="../.gitbook/assets/image (34).png" alt=""><figcaption><p>Sliver client and server.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (36).png" alt=""><figcaption><p>Sliver connected.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (37).png" alt=""><figcaption><p>Beacon generated.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (38).png" alt=""><figcaption><p>Second ngrok link generated to host malicious file download.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (39).png" alt=""><figcaption><p>Sliver listener started for beacon detonation.</p></figcaption></figure>

**Delivery concept:** The plan was to have the beacon.exe to automatically download when the user pressed "Sign In". This was tested on my phone and worked.

<figure><img src="../.gitbook/assets/image (40).png" alt=""><figcaption><p>Beacon file download successful on phone.</p></figcaption></figure>

I wasn't able to test actual detonation,the Windows VM set aside for this refused to boot, and my whole laptop crashed so the beacon callback and post exploitation side of the chain is built but unverified end-to-end.

### Overall Takeaways

Overall I learnt a lot about how phishing campaigns are created, scheduled and monitored. I also am proud that I got the C2 server setup to work. Some things I noted and probably would do differently next time:

* FIghting with Oracle's free-tier account cost more time than any part of the actual GoPhish/Sliver setup. Worth having a backup hosting plan (or just defaulting to a disposable local box + tunnel for lab work) rather than assuming cloud free tiers will "just work."
* Anything built in a modern JS framework won't clone cleanly with GoPhish's Import Site. Plan for a manual HTML rebuild step rather than expecting it to work out of the box.
* ngrok free tier is fine for short-lived lab campaigns but the rotating random subdomain (and free-tier session limits) is a real constraint compared to a stable domain. Would be a limitation if this were a real, longer engagement.
* Ensure VMs are setup so multiple can run at the same time, else everything could crash.

