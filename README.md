# The 2 a.m. Lockout

*How my personal server got locked down so thoroughly that nobody could get in, including whoever did it.*

A true story from October 2026, written up for engineer friends. Some details are left vague on purpose.

---

## The setup

I run a personal Linux server where I do most of my coding. It runs **[T3 Code](https://github.com/pingdotgg/t3code)**, a web app for driving AI coding agents (Claude, Codex, Cursor) from any browser: my desktop, my laptop, my phone, even my Tesla. It also hosts **Alfred**, my homemade assistant, and a bot that connects my coding agents to Slack.

The important detail is that T3 has a terminal built into the browser. Anyone signed in to T3 effectively has a shell on the box, and my account can run admin commands without a password. **A browser login cookie is root access with extra steps.**

## 2:27 a.m.: someone sits down at a keyboard

At 2:27 a.m., a T3 session in **Microsoft Edge on Windows** reconnected. I don't use Edge. That session had been signed in for two days, from a one-time sign-in link. Which machine opened that link in Edge is still a mystery.

For the next 30 minutes, someone did this:

**2:35–2:51, typing by hand in the T3 terminal.** The shell history includes gems like `CLEAR` and `cd /.ssh`. Then:

```
sudo rm -f /etc/ssh/ssh_host_*
sudo rm -rf /etc/ssh                          # yes, the whole directory
sudo apt-get install --reinstall openssh-server
```

The reinstall **failed**. Debian's package won't recreate a config file you deleted, because it assumes you deleted it on purpose. SSH stayed dead for the next **12.5 hours**.

**2:42, they start a Cursor AI agent** in the same terminal and give it instructions, typos preserved:

| Prompt | What the agent did |
|---|---|
| *"show all t3 code sessions and paired devices and i want to unpiar all but one"* | Signed out 11 sessions, including the Slack bot's and Alfred's |
| *"t3 sends notifications through google play services stop those"* → *"delete and kill all"* | Wiped push notifications for my phone |
| *"kill the alfred and slack processess"* | Done |
| *"delete all old ssh keys everywhre for this register new"* | Deleted the list of keys allowed to log in |
| **2:57:** *"logout of t3 account"* | Signed out **the very session they were typing in** |

Final state: zero valid sessions, no SSH server, and no SSH keys. Nobody could get in, including whoever did it.

## The accidental perfect crime

This is my favorite part.

Every one of my devices still had an open T3 tab holding a now-dead cookie, and they all started retrying nonstop. That produced about **354,000 "session token revoked" warnings**, which filled the system log until it rotated. **Everything before about 7 a.m. was gone.**

The web server for the main T3 domain wasn't keeping its own request log either. So the one fact that would answer the question, *which IP address did the 2:27 a.m. session come from*, was erased by my own devices knocking on the door.

## Morning: "Why does everything want a token?"

- Every device asked me to sign in again, and SSH refused me.
- I got back in through my hosting provider's emergency serial console. It took six failed attempts, partly because I typed the wrong username.
- **Red herring #1:** Cursor's dashboard showed AI usage around 11:30 that I hadn't made. That turned out to be **my own robot**: a daily cron job at 11:30 **UTC** that test-calls every AI model to check that my routing still works. It even cleans up its own test chats.
- **Red herring #2:** I first guessed the usage was a long run of a space-shooter game I'm building. That run used Claude, not Cursor, and had finished at midnight.

## The investigation

I had Claude send three AI agents in parallel: one on the server, one on Cursor usage, one on T3 sign-ins. Their findings:

- **No outside-intruder fingerprints.**
  - No new user accounts, no planted cron jobs or services.
  - Core system programs check out as unmodified.
  - About 5,900 failed SSH logins, which is normal internet noise.
- **It looks like a panicked lockdown, not an attack.** An attacker adds their own key and keeps access. This person:
  - rotated keys to one that only the server itself held
  - killed notifications
  - logged themselves out

  The Cursor agent even blanked out secret values whenever it read config files. Nothing was copied off the box.
- **It was someone who knew my setup closely.** They knew about Alfred, the Slack bridge, and that my phone gets push notifications through Google Play. Alfred's push notifications had only gone live three days earlier, and a coding job finished around midnight, so my phone may well have been buzzing that night.
- **The Edge clue:** the session was legitimately signed in two days earlier, through a one-time sign-in link. Nothing on the server says which Windows machine opened it, and the logs that might have told us were the ones flooded away.

**Status: unsolved.** The candidates aren't great: a sleepwalking me, someone else in the house, or a Windows machine I don't know about.

## Plot twists during the cleanup

- **A Utah datacenter IP** (a host popular with VPN services) had five open connections to the server while we were investigating. It's now firewalled.
- **Someone opened the T3 sign-in page using Tor Browser** that afternoon. They got a 401, never tried a token, and left.
- **A URL scanner fetched the exact private URL of my space-shooter conversation.** I never shared it. It got the login page and nothing else, but how it knew that URL is still a mystery.
- **The scariest find came last.** An SSH key I don't recognize had logged into my account **six times over the previous two weeks**. The first visit tried a misspelled username before the right one: a human making a typo, not a bot. Ironically, the 2:30 a.m. key purge is what removed it.

## The AI also made mistakes while fixing it

- Claude checked a new web-server config while running as root. The check created a log file owned by root, the web server couldn't open it, and **everything went down for about 2 minutes**.
- Later, it tried to reload a web server whose admin API is disabled. The reload failed, so the next command never ran, and the new IP filter **didn't apply to the main T3 site for about a minute**. Claude only caught it by testing from outside the server.

## Where things stand now

- **SSH:** accepts one key, from one IP.
- **T3 sessions:** all revoked, and each device re-paired with a labelled link.
- **Network filter:** T3 only answers my networks. Everyone else gets a `403`.
- **Fun fact:** the Tesla connects through Cisco's IoT cellular network, so the car gets its own entry in the allowlist.
- **Alfred:** its built-in browser skipped my network filter and went straight out over cellular, so Alfred now opens T3 in Chrome instead.
- **Logs:** request logging has its own file, and T3's log output is rate-limited, so a retry flood can't wipe the evidence again.
- **Alerts:** I get a Slack DM whenever a new device pairs or a session appears from a new IP.

## Lessons

1. **A browser cookie that grants a shell is a root credential.** Give it short expiry and an IP allowlist.
2. **Logs that share storage with app warnings can be wiped by those warnings.** Use a separate file and rate limits.
3. **`rm -rf /etc/ssh` survives a reinstall.** The package won't recreate a config file you deleted, on purpose.
4. **Validate configs as the service user, not root.**
5. **An AI agent will happily lock you out of your own server if you ask nicely,** typos and all.
6. **Check what timezone the dashboard is in** before you panic about 11:30.
