# CyberChallenge.IT-2026-Attack-Defense-CTF-Operations

🦅 From Albania, with exploits.

Today our team competed in CyberChallenge.IT 2026 — one of Italy's most prestigious cybersecurity simulations, organized by CINI Cybersecurity National Lab, led by Gaspare Ferraro. An Attack & Defense CTF alongside Italian ECSC champions and teams from Kosovo.

Albania Team #85. A team with a plan. A team with roles. A team that hit #1.


📞 THE CALL — BEFORE EVERYTHING


Before a single line of code was written, before VPN was configured, before the competition even opened — we got on a Discord call.

Roles were assigned:

🔴 Red Team / Exploit Crafter — my role
Vulnerability analysis, exploit development, and flag capture. Emergency fallback: if a service went down mid-competition, switch immediately to patching.

🛡️ Patching Team
Standby to receive source code and vulnerability reports from Red Team and deploy fixes as fast as possible.

🔵 SOC (x4 members)
Monitoring Tulip traffic in real time, detecting incoming attacks, analyzing PCAP flows, and flagging suspicious activity.

This was not improvisation. This was structured offense and defense.


🔍 DAY BEFORE — RECONNAISSANCE


The day before the competition, I ran a single curl command against the gameserver's public endpoint:

  curl http://10.10.0.1:8081/flagIds

The response contained the full team list — IDs, shortnames, and enough metadata to map every enemy team to their vulnbox IP using the pattern 10.60.{team_id}.1.

86 targets. Mapped. Before the game even started.

This is not a vulnerability — it's the intended design. But in Attack & Defense, preparation is offense.


🔧 INFRASTRUCTURE — BUILT AS A TEAM


The team deployed and configured the full operations stack together:

🖥️ VPS (DigitalOcean)
Central operations server. All tools hosted here, accessible by every team member.

🔒 WireGuard VPN
Secure tunnel to the competition network.
Config: player010.conf | Endpoint: 167.235.246.128:51820

📡 Tulip — Traffic Analyzer
Open-source CTF traffic analyzer. Captured and indexed all PCAP traffic from the vulnbox in real time. The SOC team's primary tool — monitoring incoming exploits, identifying attack patterns, reverse-engineering enemy techniques from raw network flows.

🚩 DestructiveFarm
Open-source automated flag submission framework. Configured to submit captured flags to the gameserver every tick with the team token.

🗂️ LessDestructiveFarm
Lightweight flag viewer for real-time submission status and history.

🛠️ Albania CTF Control Panel (custom built)
A full web dashboard developed specifically for this competition — live service status, SSH console in browser, PCAP sync pipeline (tcpdump → rsync → Tulip), flag tracking, enemy team map with all 86 IPs, backup and firewall management.

Every tool was live and tested before the first slot opened.


⚔️ GRACE PERIOD — 30 MINUTES, EVERY SECOND COUNTS


The moment SSH access opened, the clock started.

The first thing I did was not analyze, not explore — it was backup.

  tar czf /tmp/backup_$(date +%H%M%S).tar.gz /root/ /home/ /opt/

3 minutes. That's how long the backup took.

But those 3 minutes were a calculated investment:
→ The backup gave the Patching Team a safe snapshot of every service
→ They could study the source code from the backup without touching live services
→ If a patch broke something, we could restore in seconds
→ The Patching Team had 27 minutes to work — while I pivoted immediately to exploit development

The remaining 27 minutes of the grace period:
✅ Enumerated all 4 running services via docker ps
✅ Pulled full source code of every service
✅ Identified vulnerabilities through manual source code analysis
✅ Started tcpdump traffic capture → /opt/pcap/
✅ Wrote 3 exploit scripts — before the network opened

While I was building exploits, the Patching Team was analyzing the backup and the SOC team was already live on Tulip — ready the moment the first packet hit.

30 minutes. Three parallel workstreams. Zero wasted time.


🚩 THE SERVICES & VULNERABILITIES


The vulnbox ran 4 services. Here is what the Red Team found and exploited:


1️⃣ SeaOfHackerz — Port 5050
Vulnerability: SQL Injection via raw Python string formatting

The login function used % string formatting instead of parameterized queries:

  cur.execute("SELECT * FROM users WHERE username = '%s'
               AND password = '%s'" % (username, password))

Sending password = "a' OR '1'='1' --" bypassed authentication entirely. Flags were stored in inventory.personal_description, returned by GET /api/user/items.

Exploit flow:
→ Register a new user
→ GET /api/users — enumerate all usernames (paginated)
→ SQL inject for each username → bypass login
→ GET /api/user/items → extract flag from personal_description

Result: 366+ flags | +5,029 attack points


2️⃣ GadgetHorse — Port 3000
Vulnerability: IDOR via Cookie Manipulation

The cart cookie was simply base64-encoded JSON with no server-side ownership verification:

  cart = base64( JSON([{"id": productId, "qty": 1}]) )

By encoding any productId from the Flag IDs endpoint into the cart cookie, I could retrieve any user's custom sticker — which contained the flag — without authentication.

  GET /custom-sticker/{productId}
  Cookie: cart={base64_encoded_payload}

Result: 105+ flags | +1,760 attack points


3️⃣ CheesyCheats — Port 5000/5555 (gRPC + TLS)
Vulnerability: Diffie-Hellman Authentication Bypass

This service used a custom DH key exchange over gRPC with TLS. The implementation had a critical mathematical flaw:

  If g_a = 1:
  K = pow(1, b, p) = 1   for ANY value of b

The server stores K = '1' in the login keys. Send K = '1' in LoginStep2 — and you receive a valid session token for any user. No password. No brute force. Pure math.

Additional vulnerabilities identified:
— Poly1305 Key Reuse: same TOKEN_KEY for all tokens → MAC forgery possible
— MAC over md5(ciphertext) instead of raw ciphertext → weak integrity
— LoginKey Replay: invalidation logic flawed

Exploit flow:
→ LoginStep1(username=TARGET, g_a="1") → server stores K="1"
→ LoginStep2(username=TARGET, K="1") → valid session token
→ Redeem(cheat_id=FLAG_ID) with auth-token → flag

Tools: grpcio, grpcio-tools, TLS with ca-cert.pem extracted from vulnbox


4️⃣ MineCClicker — Port 9999
Binary service over TCP (socat). SLA issues detected (~92%) but full exploitation was not completed within the competition window.


📊 PEAK PERFORMANCE


When the network opened and the exploits hit:

🥇 Albania Team #85 — RANK #1

Above Italian university teams.
Above Kosovo teams.
Above 86 competitors.

The SOC team was live on Tulip, monitoring every flow.
The exploits were running across all 86 targets, every tick.
Flags were landing in real time.

Then came the message from the organizers:

❝ 5,500 new connections in a single round. ❞

The Red Team's exploits were hitting all 86 enemy teams simultaneously with no rate limiting — generating thousands of connections per tick that the competition infrastructure couldn't absorb. We were banned for inadvertent DoS.

The fix is two lines of code:

  import time
  time.sleep(2)  # between each target

The ban held for the remainder of the slot. We watched the scoreboard from the outside.

The #1 was ours. The lesson belongs to all of us.


💡 LESSONS LEARNED


▸ SQL Injection is still alive and critical in 2026 — parameterized queries are not optional
▸ Custom cryptography breaks at the edges — always validate g_a = 0 and g_a = 1 in DH implementations
▸ IDOR vulnerabilities hide in unexpected places — not just URLs, but cookies and headers too
▸ In Attack & Defense, rate limiting your exploits is as important as writing them
▸ A 3-minute backup at the start saves 27 minutes of structured parallel work
▸ Role separation works — SOC + Red Team + Patching Team is the right structure
▸ Infrastructure preparation before the competition is half the battle



Thank you to CINI Cybersecurity National Lab and Gaspare Ferraro for organizing this simulation and for including Albania. Competing alongside Italian ECSC champions was a privilege.

To the team — the Discord call, the roles, the SOC work, the patching, the infrastructure — this was built together.

Albania was in the room. 🦅🇦🇱

#CyberChallenge #CTF #AttackAndDefense #Cybersecurity #Albania #RedTeam #SOC #PenTesting #SQLInjection #IDOR #DiffieHellman #gRPC #ExploitDevelopment #CyberSecurity2026 #CINI #WireGuard #Tulip #DestructiveFarm #InfoSec #TeamWork
