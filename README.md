# linkedin-connections-at

A [Claude Code](https://claude.com/claude-code) skill that answers one question: **"Who in my LinkedIn network works at company X?"**

Give it a company name. It uses your own logged-in Chrome to find:

- your **1st-degree** connections who work at that company,
- optionally your **2nd-degree** people there, with the **mutual connections** you could ask for an intro,
- optionally **former** employees,
- optionally **who at that company one of your connections knows**.

It is read-only. It never sends messages, connection requests or follows.

```
/linkedin-connections-at Acme
/linkedin-connections-at Acme 1st and 2nd degree
```

Plain requests such as "who do I know at Acme?" should also work, since the skill's description matches them.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [BrowserSkill](https://github.com/Tencent/BrowserSkill) (`bsk` CLI plus its Chrome extension), which lets the agent use your real, logged-in browser
- Chrome, logged in to LinkedIn

Tested on macOS with Chrome. Not tested on Windows or Linux; the skill's shell snippets assume a POSIX shell.

## Install

**1. Install BrowserSkill** (see its [install guide](https://github.com/Tencent/BrowserSkill/blob/main/AGENT_INSTALL.md) for details and other platforms):

```bash
curl -fsSL https://raw.githubusercontent.com/Tencent/BrowserSkill/main/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
bsk --version
```

**2. Install BrowserSkill's own skill for Claude Code.** Check the harness ID first:

```bash
bsk install-skill --list --json
bsk install-skill --harness <id-from-the-list>
```

**3. Install and connect the Chrome extension** by following
[step 4 of the install guide](https://github.com/Tencent/BrowserSkill/blob/main/AGENT_INSTALL.md#4-connect-the-browser-extension), then check:

```bash
bsk doctor    # "extension connected" should be ok
```

**4. Install this skill:**

```bash
git clone https://github.com/ahiraz/linkedin-connections-at ~/.claude/skills/linkedin-connections-at
```

Start a new Claude Code session, make sure you are logged in to LinkedIn in Chrome, and run the command above.

## How it works

1. Looks the company up in LinkedIn's company search and checks the name matches (LinkedIn sometimes auto-corrects names).
2. Reads the company's LinkedIn IDs (parent company and sub-entities) from its "employees" link.
3. Runs a people search filtered by those IDs and by connection degree, and reads the result cards.
4. Reports a table of names, headlines, profile URLs and, for 2nd-degree, the named mutual connections.

A handful of page loads per run. No bulk crawling of profiles.

## Limitations

- It only sees employment that people **list on their own profile**. Advisors, contractors and unlisted roles won't appear.
- 2nd-degree results are the top 20 in LinkedIn's own order. LinkedIn shows no total, and only two mutual connections are named per person.
- "Zero results" can also mean someone hides their connections list.
- LinkedIn changes its pages often. If a step stops returning data, open an issue.

## Disclaimer

LinkedIn's User Agreement restricts automated access to its site. This skill is read-only and low-volume and runs on **your own account in your own browser**, but using it is still automation and could lead LinkedIn to restrict your account. Use it at your own risk and only on your own network data. This project is not affiliated with LinkedIn, Anthropic or Tencent.

## License

[MIT](LICENSE)
