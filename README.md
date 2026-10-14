# Telegram Remote Administration Tool (Educational)

> A hands-on study project that implements a Telegram-controlled remote
> administration tool for Windows, built to learn **how remote-access tooling
> works and, more importantly, how defenders detect it**.

![Purpose](https://img.shields.io/badge/purpose-educational-informational)
![Use](https://img.shields.io/badge/use-authorized%20lab%20only-critical)
![Language](https://img.shields.io/badge/python-3.8+-3776AB?logo=python&logoColor=white)
![Platform](https://img.shields.io/badge/platform-Windows-0078D6?logo=windows&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue)

---

> ## ⚠ Authorized, lab-only use
>
> This project is published for **education and defensive research**. Run it
> **only** on a machine you own, or one you have **explicit written permission**
> to test, inside an isolated lab.
>
> Installing or operating remote-administration software on a computer you do
> not own and are not authorized to control is illegal in most jurisdictions
> (for example, unauthorized-access and computer-misuse laws). **You are solely
> responsible for how you use it.** The author provides it with no warranty and
> accepts no liability for misuse. If you are not sure whether your use is
> authorized, do not run it.

---

## Why this project exists

Remote Access Tools (RATs) are a standard part of the malware landscape, and
understanding them is part of a defender's job. Building a small, readable one in
a lab is one of the clearest ways to learn:

- how a command-and-control (C2) channel is structured,
- which operating-system facilities attackers abuse, and
- **what those actions look like from a defender's seat** so you can detect them.

The last point is the real goal here. Every capability below is paired with how
a blue team would catch it (see [Detection & defense](#detection--defense)).

## What it is

A Python application that runs on a Windows PC and is driven through a
[Telegram bot](https://core.telegram.org/bots). Instead of a custom C2 server,
it uses Telegram's Bot API as the control channel: the operator sends menu
button presses in a Telegram chat, the program on the PC acts on them and replies
with text, files, or screenshots. Access is restricted to a single configured
administrator Telegram ID.

```
   Operator (Telegram app)                Telegram Bot API                 Controlled PC (Windows)
 ┌────────────────────────┐            ┌──────────────────┐            ┌────────────────────────────┐
 │  taps menu buttons      │  HTTPS     │  api.telegram.org │   polling  │  python bot process         │
 │  /start  /help          │ ─────────► │   (relay)         │ ◄───────── │  pyTelegramBotAPI long-poll │
 │  receives text / files  │ ◄───────── │                   │ ─────────► │  executes handler, replies  │
 │  / screenshots          │            └──────────────────┘            │  os / ctypes / psutil / mss │
 └────────────────────────┘                                             └────────────────────────────┘
```

Using a public messaging service as the relay is itself an instructive detail:
it is why this family of tooling blends into normal HTTPS traffic, and why
network-only detection is hard (see the defense notes).

## Capabilities and ATT&CK mapping

Each module maps to one or more [MITRE ATT&CK](https://attack.mitre.org/)
techniques, the industry-standard catalog of adversary behavior.

| Module | Capability | ATT&CK technique |
|--------|------------|------------------|
| `info.py` | Host name, OS, CPU, and public IP lookup | T1082 System Information Discovery · T1016 System Network Configuration Discovery |
| `info.py` | Screenshot capture (via `mss`) | T1113 Screen Capture |
| `files.py` | Browse the filesystem and send files to the chat | T1083 File & Directory Discovery · T1005 Data from Local System · T1041 Exfiltration Over C2 Channel |
| `cmd_mode.py` | Run arbitrary shell commands | T1059 Command and Scripting Interpreter |
| `processes.py` | List running processes / terminate a process | T1057 Process Discovery · T1489 Service Stop |
| `remote_input.py` | Inject keystrokes and shortcuts (via `pyautogui`) | T1565 Data Manipulation (input) |
| `power_control.py` | Lock, shut down, or reboot the host | T1529 System Shutdown/Reboot |
| `power_control.py` | Add/remove a Startup-folder launcher | T1547.001 Boot or Logon Autostart Execution |
| transport | Telegram Bot API as the C2 relay | T1102 Web Service · T1071.001 Application Layer Protocol: Web |

## Repository layout

```
.
├── info.py           host info + screenshot handlers
├── files.py          Telegram file-browser (paged keyboard)
├── cmd_mode.py       shell-command handler
├── processes.py      process list / kill handlers
├── remote_input.py   keystroke / shortcut injection handlers
├── power_control.py  lock / shutdown / reboot / startup handlers
├── keyboards.py      Telegram reply-keyboard (menu) definitions
├── requirements.txt  Python dependencies
└── README.md
```

> Note: these are the handler modules. Wiring them together needs an entry
> point that creates the bot, loads configuration, and registers the handlers
> (see setup).

## Setup (for your own lab machine)

**Prerequisites:** Windows, Python 3.8+, and a PC you own or are authorized to use.

1. **Clone and install dependencies**
   ```bash
   git clone https://github.com/securewithharshitrajput/RAT.git
   cd RAT
   pip install -r requirements.txt
   ```

2. **Create a Telegram bot.** Message [@BotFather](https://t.me/BotFather),
   follow the prompts, and copy the HTTP API **token** it returns.

3. **Find your Telegram user ID** (used to restrict access to only you), for
   example with [@userinfobot](https://t.me/userinfobot).

4. **Configure.** Provide the token and your admin ID via a `config.ini`:
   ```ini
   [TelegramBot]
   token = your_bot_token

   [Admin]
   id = your_numeric_telegram_id
   ```
   Only the configured admin ID is allowed to drive the bot.

5. **Run it on your lab PC**, open your bot in Telegram, and send `/start` to get
   the control menu.

## Dependencies

`pyTelegramBotAPI` (Telegram bot framework), `keyboa` (inline keyboards),
`psutil` (process info), `mss` (screenshots), `requests` (HTTP), plus their
transitive packages. See [`requirements.txt`](requirements.txt).

## Detection & defense

This is the part worth studying. If you can build it, you should be able to catch
it. Here is how each behavior surfaces to a defender.

| Behavior | What a defender watches for |
|----------|-----------------------------|
| **Telegram C2** | A non-browser process (here, `python.exe`) making repeated HTTPS connections to `api.telegram.org`. Baseline which hosts legitimately talk to Telegram; flag the rest. |
| **Startup persistence** | Creation of a `.bat` in the user's `Startup` folder, or new Run-key entries. File-integrity monitoring and autostart auditing catch this. |
| **Screen capture / input injection** | An unexpected Python process loading `mss`/`pyautogui` and capturing the screen or synthesizing input. EDR behavioral rules and parent-process lineage help. |
| **Arbitrary command execution** | `python.exe` spawning `cmd.exe`/`powershell.exe` children, or calling `wmic`, `shutdown`. Process-ancestry alerts (unusual parent → shell) are high-signal. |
| **Process termination** | Mass or targeted `terminate()` calls, especially against security tooling. |

**Mitigations:** least-privilege accounts, application allow-listing, egress
filtering and DNS monitoring, EDR with behavioral detections, and autostart/
registry auditing.

### Tie-in: detect it with kguard

The process-launch, command-execution, and network-connection behaviors above
are exactly what an eBPF endpoint monitor observes. The companion project
[**kguard**](https://github.com/securewithharshitrajput/kguard) ships
MITRE-mapped rules for several of them (for example, a shell spawned by an
unexpected parent, or a process making an outbound connection). Writing a rule
that flags *this* tool with *that* monitor is an excellent end-to-end exercise:
build the behavior, then detect it.

## Credits

Built as a learning exercise on top of the Telegram bot ecosystem:

- [pyTelegramBotAPI](https://github.com/eternnoir/pyTelegramBotAPI)
- [Keyboa](https://github.com/torrua/keyboa)
- Inspired by the open-source `telegram-remote-bot` project.

## License

Released under the MIT License for **educational use**. The ethical and legal
responsibility for any use rests entirely with the user. See the lab-only notice
above.
