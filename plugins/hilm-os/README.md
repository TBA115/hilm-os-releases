# Hilm OS plugin for Claude Code

The plugin is built and checked in Hilm's backend repository and published from there to the public `TBA115/hilm-os-releases` repo, which is where people install it from. It holds no secret: each person's token comes in through `userConfig` at install.

## What it does, and where

It lets a person's Claude Code read the Hilm OS team board and keep it up to date while they work on a Hilm task: who is on what, a warning when two people are on the same task, one line of progress, and the sub tasks of a task.

**It works in Hilm repositories only.** People also use Claude for their own projects and for other work, and none of that is Hilm's business. Three things keep it that way:

1. **It is switched on one folder at a time.** The install line below is run inside a Hilm repo's folder with `--scope local`, and Claude Code then loads the plugin in that folder only. Anywhere else the plugin is not there: no hooks, no tools.
2. **The hooks check the folder before they speak.** A hook first reads the `origin` address of the git repository the folder belongs to, the one it was cloned from. Only a `github.com` address under `HILM-Technologies`, or the Hilm OS repos themselves (`TBA115/hilm-os` and `TBA115/hilm-os-...`), lets it call the server. From any other folder the hooks send nothing at all. The hook that spares a question for the reading, starting and progress tools makes the same check, so outside a Hilm repo it lets nothing through: Claude Code's own permission rules decide there, as for any tool.
3. **The server refuses the rest.** Every tool call has to name a Hilm repo, and a call that names none is refused before the board is read or anything is written. A request that names no Hilm repo leaves nothing in Hilm OS's database: no line, no time of use. Like any request it shows in the server's request log, and it counts for a minute toward the limit on calls. A note or a sub task name that looks like it holds a key or a password is refused too. The server takes the repo's name as Claude gives it: it cannot see which folder a call came from, which is why the per-folder install comes first.

**Why `--scope local` matters.** Points 2 and 3 are a second and a third line of defence, not a licence to install it for the whole computer. Installed that way, Claude Code itself opens the plugin's connection to the board at the start of every session, in any folder, to list the tools. The hooks stay silent and Hilm OS records nothing, but the request still reaches the server and its request log, with its time and network address. Only the per-folder install prevents that. If it was installed for the whole computer, take that install away with `claude plugin uninstall hilm-os@hilm --scope user`.

A folder counts only by the `origin` of the git repository it belongs to, and the origin must have exactly one address. So a folder inside a clone of a Hilm repo counts as that repo, whatever is kept in it, unless it is a git repository of its own. A fork under a person's own account, a Hilm repo added as a second remote of a private project, and an address written with an SSH host alias (`git@work:HILM-Technologies/...`) do not count: Claude stays silent there. `git config --local --get-all remote.origin.url` shows what the hooks read: it has to print exactly one line, a github.com address of a Hilm repo.

**What leaves the laptop:** which task was started or finished, a new task if Claude makes one, the one line Claude writes about a task and whether the person is blocked, the names of sub tasks Claude adds and which ones it ticks, the name of the Hilm repo the session is in, and, at each session start, prompt and tool call in a Hilm repo, the fact that Claude is in use there (the server keeps the repo and a time within a minute of the latest one: it writes at most once a minute, or at once when the repo changes). **What never does:** prompts, files, folder names, or anything from a folder that is not a Hilm repo. Claude Code hands every hook the person's prompt; these hooks do not read it. The words that do leave are the ones Claude writes about a task (a title, a description, one-line notes, the names of sub tasks) and the words it searches the board for, which are not kept. Claude adds and ticks sub tasks as it starts a task or reports progress, so in a Hilm repo they go without asking each time, like the note. Claude is told to keep code, logs, file contents and personal details out of its notes and sub task names, and the server refuses a title, a description, a note or a sub task name that looks like a key or a password. Nothing else checks them.

Everyone on the team can read the tasks, their sub tasks and the notes on the Board, with how long ago each was written. Each person sees the last 30 lines their own Claude sent in the Hilm OS app, in Settings, Connect Claude. Whether a person's Claude is connected (heard from a Hilm repo in the last 14 days) is shown to the whole team. Only the person sees which repo it was last used in and at what time; the founder also sees the day, not the time and not the repo.

## Install

It needs the `claude` command on PATH. The desktop app and the VS Code extension alone don't add one, so install the CLI too. Use Claude Code 2.1.139 or newer, or 2.1.268 or newer if anyone runs `claude -p`, because `PermissionRequest` hooks don't fire there before that. It needs `git` and `curl` on PATH, which every Hilm laptop has: any version of curl will do.

Make a token in the Hilm OS app, in Settings, Connect Claude. The page shows these lines with the token filled in.

Once per computer, from any folder:

```bash
claude plugin marketplace add TBA115/hilm-os-releases
```

Then in a terminal **inside a Hilm repo's folder**:

```bash
claude plugin install hilm-os@hilm --scope local --config token=hos_...
```

In each other Hilm repo, in its folder (the token is already stored, once per computer):

```bash
claude plugin install hilm-os@hilm --scope local
```

Start Claude in that folder. It says `Hilm OS: connected as (name), in (repo)`.

Never install it without `--scope local` (see "Why `--scope local` matters" above).

The install line holds the token, and a terminal keeps a history of the lines typed into it. The token only works with the board, lasts 90 days, and can be ended at any time in the app. Do not paste the line anywhere else.

To renew an ended token, make a new one and run the second line again in any Hilm repo's folder. It answers that the plugin is already installed, and the new token takes effect all the same. If the token has ended, the start of each session in a Hilm repo shows the person one line saying so, and the app's Connect Claude page shows Not connected, unless another computer of theirs is still connected. If the token was lost on the computer, for example after `claude auth logout`, which deletes plugin settings, the start of each session says there is no usable token; the page goes on showing the day Claude was last heard, and says Not connected once 14 days have passed without it. Either way, make a new token and run the install line again.

## Switch it off

In one folder, run this in a terminal in that folder:

```bash
claude plugin uninstall hilm-os@hilm --scope local
```

On a whole computer: end every token made for it, in the Hilm OS app, in Settings, Connect Claude. Nothing works with an ended token. The plugin itself stays in each folder until the line above is run there: until then it goes on asking Hilm OS, which refuses it, and the start of a session in that folder says Hilm OS did not accept the token.

## Update

Plugins from a third-party marketplace don't update themselves. When the team says there is a new version, run this once per computer, from any folder:

```bash
claude plugin marketplace update hilm
```

Then this in each Hilm repo's folder where the plugin is on:

```bash
claude plugin update hilm-os@hilm --scope local
```

It takes effect in the next session. The line names `--scope local` because without it older Claude Code looks for a whole-computer install and answers that the plugin is not installed.

## How the hooks are built

`hooks/hooks.json` is not written by hand. It is built in Hilm's backend from the server's own list of Hilm addresses, and a test there fails if the file and the list ever differ. The same test runs the hooks for real, on Windows and on Linux, in folders with many kinds of remote address, Hilm's and not.

- **`git` runs the hook.** Each hook is `git -c alias.hilm-os=!<one line of sh> hilm-os <event>`: one program and its arguments, so no shell line for Windows to misread, and git's own `sh` runs the line wherever git is installed, with no Git Bash on PATH.
- **The folder check comes first.** `git config --local -z --get-all remote.origin.url` reads only the settings of the repository the folder belongs to, never the person's own git settings for the whole computer, and `origin` must have exactly one address: with two or more the hook cannot tell which one git uses, so the folder is not counted. The address has to be GitHub's, whole, in a form git writes, and the repo's name may hold only letters, digits, dot, dash and underscore, so nothing from an address can reach the request except an `owner/name`. An address written to look like two lines stays one line and does not match. It is the address as the folder records it: a url rewrite (`insteadOf`) is not looked at. No match, no request.
- **The token is on no command line.** It goes from `CLAUDE_PLUGIN_OPTION_TOKEN` to curl on curl's standard input (`-K -`), so it is in no process list and no log of command lines. It does sit in the environment of the hook's own processes, where Claude Code puts it. That pipe is also why the prompt, which sits on the hook's own standard input, is never read. A token that does not look like one (anything but letters, digits, dash and underscore) is not sent at all: curl would read a line break in it as more settings.
- **`curl -q` comes first.** The person's own curl settings file is not read. With `include` in it the answer's headers would reach Claude, and with `location` a redirect would be followed, token and all.
- **Only a whole HTTP 200 answer reaches Claude.** curl itself has to end well, and the body has to begin with `{` and end with `}`, so an answer cut short by the time limit is dropped. A sign-in refusal becomes one plain line of ours, and a Railway or proxy error page, or half an answer, becomes another. At the start of a session the person is shown that line (`systemMessage`, which Claude itself is not given); on a later prompt it goes to the hook's error output only, so it is said once a session. Neither is ever shown to Claude as if Hilm OS had said it.
- **It never ends with 2.** A `UserPromptSubmit` hook that ends with 2 refuses the prompt, so a lost token or a server that is away never costs a prompt. Past the folder check the line always ends with 0. In a folder whose git settings are broken, git itself stops first with its own error, which is not 2 either, and nothing is sent.
- **`--max-time 3`:** a stalled backend costs about 3 seconds a prompt, in Hilm repos only. `timeout` stays as a backstop.
- **`PermissionRequest`, never `PreToolUse`:** it answers only when a prompt is about to appear, so Manual mode doesn't stall on a progress report and auto mode keeps its classifier. It makes the same folder check as the other two hooks and calls no server: in a Hilm repo it lets reading, starting and one line of progress through unasked, with the sub tasks those two add and tick, and anywhere else it says nothing, so Claude Code's own permission rules decide, as for any tool. `finish_task`, `create_task` and `assign_task` aren't listed, so this hook never lets them through: they are asked about unless the person's own Claude Code settings allow them.
