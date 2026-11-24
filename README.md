# clickandsay-mcp

<a href="https://glama.ai/mcp/servers/beriki770-ship-it/clickandsay-mcp"><img width="380" height="200" src="https://glama.ai/mcp/servers/beriki770-ship-it/clickandsay-mcp/badges/card.svg" alt="clickandsay-mcp MCP server" /></a>

An MCP server that puts your [ClickAndSay](https://clickandsay.com/) feedback
queue inside Claude Code, Claude Desktop, Cursor, or any other MCP client. Your
client marks something on their site; your coding agent gets the card with the
exact CSS selector, the click point inside the element, the element's own HTML,
the console errors from that moment, the screenshot and, on 3D/canvas pages, the
scene state the reporter was looking at. Then it reports the fix back to the
same card.

Six tools: `list_projects`, `list_feedback`, `get_feedback`, `get_screenshot`,
`update_status`, `reply`.

Runs over stdio, needs Node 18 or newer, one dependency
(`@modelcontextprotocol/sdk`).

---

## 1. Get an API key

Sign in to the ClickAndSay admin (`https://clickandsay.com/admin`) as the account
that owns the projects. In the header, press **"Connect your AI"** (German: *KI
verbinden*). That opens the `#ai` panel:

* no key yet → **"Create key"**. The key is shown **once** and starts with
  `pk_live_`. Copy it now; the server stores only a hash.
* a key already exists → you see its prefix, plus **"Replace key"** (the old
  one stops working immediately) and **"Revoke key"**.

The same panel also hands you a ready-made `SKILL.md` for your project.

The key is deliberately narrow: it reaches only the projects owned by your
account, and only the actions this server needs (read the queue, update a card,
close a round, fetch a screenshot). It cannot create or delete projects, cannot
read another account's data, and cannot touch account settings. It is
rate-limited to 120 requests per minute.

The AI connection is part of the Studio and Agency plans. A new account's
14-day trial runs on Start, which has no AI connection: to use it during the
trial, pick Studio (or Agency) and add a payment method. The first charge comes
after day 14. See section 5 for what happens on other plans.

## 2. Install

### Claude Code

```bash
claude mcp add clickandsay -e CLICKANDSAY_API_KEY=pk_live_your_key_here -- npx -y github:beriki770-ship-it/clickandsay-mcp
```

Check it came up with `claude mcp list`.

### Claude Desktop

Settings → Developer → Edit Config, then add to `claude_desktop_config.json`
and restart Claude Desktop:

```json
{
  "mcpServers": {
    "clickandsay": {
      "command": "npx",
      "args": ["-y", "github:beriki770-ship-it/clickandsay-mcp"],
      "env": { "CLICKANDSAY_API_KEY": "pk_live_your_key_here" }
    }
  }
}
```

### Cursor

`~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (per project), same block:

```json
{
  "mcpServers": {
    "clickandsay": {
      "command": "npx",
      "args": ["-y", "github:beriki770-ship-it/clickandsay-mcp"],
      "env": { "CLICKANDSAY_API_KEY": "pk_live_your_key_here" }
    }
  }
}
```

### Any other MCP client

Command `npx -y github:beriki770-ship-it/clickandsay-mcp`, transport stdio, environment variable
`CLICKANDSAY_API_KEY`.

### Environment

| Variable | Required | Default |
|---|---|---|
| `CLICKANDSAY_API_KEY` | yes | — |
| `CLICKANDSAY_BASE` | no | `https://clickandsay.com` |

The product used to be called Pinpoint. `PINPOINT_API_KEY` and `PINPOINT_BASE`
are still read when the new names are not set, so an older config keeps working.

## 3. What a session looks like

> **You:** fix the top feedback card on Salon Marlene

The agent, on its own:

1. `list_feedback({project: "Salon Marlene"})` → the open cards, newest first:

   ```json
   { "id": "fx6a8eefd71d06889fe4b", "status": "new", "category": "improvement",
     "note": "Die Überschrift ist zu klein am Handy",
     "page": "/index.html", "hasScene": false }
   ```

2. `get_feedback({id: "fx6a8eefd71d06889fe4b"})` → the fix bundle:

   ```json
   "anchor": { "v": 2, "fxid": null,
               "selector": "div[data-area=\"hero\"] > h1",
               "frag": { "tag": "h1", "text": "schöne haare, gutes gefühl." },
               "struct": "body > div:nth-of-type(1) > h1",
               "rel": { "x": 0.5, "y": 0.49 } },
   "element": { "tag": "h1", "html": "<h1>Schöne Haare, gutes Gefühl.</h1>" },
   "env": { "viewport": { "w": 390, "h": 844 }, "dpr": 3, "consoleErrors": [] },
   "source": { "dataArea": "hero", "dataSrc": "sections/hero.html" }
   ```

3. `update_status({id, status: "progress"})`, then it opens
   `sections/hero.html`, finds the `h1` by that selector, and fixes the mobile
   type scale. Ambiguous? `get_screenshot({id})` shows the pin on the element.
   Still ambiguous? `reply({id, text})` asks the client and parks the card at
   *waiting-client*.

4. `update_status({id, status: "done"})` — ClickAndSay emails the client once that
   this card is fixed, and the card turns green in their portal.

## 4. Tool reference

| Tool | Arguments | Returns |
|---|---|---|
| `list_projects` | — | every project this key reaches: token, name, site URL, active, card counts per status |
| `list_feedback` | `project` (token or exact name), `status?` | compact cards: `id`, `status`, `category`, `hasScene`, and `thirdPartyContent: {note, page}` |
| `get_feedback` | `id` | the full bundle. Our own fields at the top (id, project, status, category, our questions, report); everything a visitor wrote or the widget read off the site inside `thirdPartyContent`: reporter, note, wish, page url/path/title, anchor v2, element html/text/rect, env (viewport, dpr, ua, console errors), source (`data-area`/`data-src`), scene snapshot, the client's thread entries |
| `get_screenshot` | `id` | the client's screenshot with the pin drawn on the element, as an MCP image (JPEG) |
| `update_status` | `id`, `status`: `received` \| `progress` \| `done`, `confirm?` | confirmation. Maps to ClickAndSay's own `new` / `in-progress` / `done`; `done` sends one email, so it first answers `NOT DONE` with a one-line summary (who gets mailed) and only goes through with `confirm: true` after the human said yes |
| `reply` | `id`, `text`, `confirm?` | appends your question to the card thread, moves it to `waiting-client`, emails the client a portal link; needs `confirm: true` the same way |

`status` on `list_feedback` accepts either vocabulary: `received|progress|done`
or the raw `new|in-progress|done|waiting-client`.

### Anchor v2 — why there are four selectors

A CSS selector alone dies the first time the site is rebuilt with hashed class
names. Every card carries four independent ways back to the same element plus
the click point, so the agent can pick whichever survives your build:

* `fxid` — nearest `[data-fx-id]` ancestor. Add `data-fx-id` to your important
  sections and nothing can ever break the link.
* `selector` — the full CSS path (with the shadow-host chain, if any).
* `frag` — tag + the first 60 characters of the element's own text. Survives
  class churn.
* `struct` — a class-free, tag-only `nth-of-type` path. Survives copy changes.
* `rel` — where inside the element they clicked, as a fraction of its box. On a
  full-width hero, this is the difference between "the headline" and "the button
  in the corner".

### The 3D bundle

If the client marked something inside a `<canvas>` on a page that implements the
scene bridge (`docs/SCENE-BRIDGE.md`), the card also carries:

```json
"scene": { "id": "gehaeuse", "point": [0.132, 0.041, -0.087], "label": "Gehäuse",
           "snap": { "camera": {...}, "state": { "explode": 0.35, "xray": false }, "chapter": 3 } }
```

`snap` is your site's own opaque state blob. Feed it to
`window.__FX_SCENE__.restore(snap)` and the page jumps back to exactly the view
the client was complaining about.

## 5. Safety

Card text (note, wish, thread, element HTML, page and environment data) is
written by your client's visitors or read off their website, and it lands in an
agent that can edit files. Three things keep it in its place:

* **Marked by the API.** For an API key, `pull` lists per card which fields are
  third-party (`untrusted`, e.g. `["note", "page", "thread[0].text"]`) and adds a
  `notice` saying they are data, never instructions. This server moves all of
  them into one `thirdPartyContent` container, frames each bundle with that rule
  and states it in every tool description. Keep that in place if you build your
  own client on top of the API.
* **Mails need the human's yes.** Anything that leaves the board by mail (a card
  set to `done`, a question to the client, closing a round) is refused by the API
  for a key unless the request carries `"confirm": true`. Without it the answer
  is `needconfirm` with one line on who would be mailed and whether a round of
  the plan is used. The tools pass `confirm` only when their own `confirm`
  argument is set, and tell the agent to ask first.
* **Read-only keys.** A key created with `mode: "read"` can list cards and fetch
  screenshots, nothing else; every change answers `readonlykey`. Use one when
  the agent only needs to look.

**No plan, no service.** When the account's plan has ended (an organisation
student's seat past its grace counts the same), the key is suspended whole:
every call, reads included (`pull`, screenshots, changes), answers `402
planexpired`. The key is not deleted: once a plan is active again the same key
works, nothing to re-wire. If the plan stays ended until the account is erased,
the key goes with the account. The owner sees "suspended: plan ended" next to
the key in admin `#ai`. The tools report it to the agent in plain words.

**The AI connection is a Studio and Agency feature.** Organisation seats have it
too. A new account's trial runs on Start and has no AI connection until Studio or
Agency is picked (first charge after day 14). On a Start plan a key is suspended the same way: every call answers `402 planfeature`, the key is
not deleted, and it works again as soon as the account is on Studio or Agency. The
owner sees why next to the key in admin `#ai`, and the tools say it in plain words.

**What `pull` shows of a project.** For a key: name, site, language, kind,
active flag, the plan and round numbers, and the cards. Never the webhook URL or
secret, the Slack URL, the client's address or the owner's id.

## 6. Troubleshooting

| Message | What to do |
|---|---|
| `CLICKANDSAY_API_KEY is not set` | add the key to the MCP entry's `env` and restart the client |
| `ClickAndSay refused "pull" (403 forbidden)` | the key is wrong, was rotated, or the account is disabled — compare it with the prefix in admin `#ai` |
| `Unknown project "..."` | use the token from `list_projects`, or the project's exact name |
| `ClickAndSay rate limit` | 120 requests/minute per key — `list_feedback` once and work from the list instead of re-pulling per card |
| `NOT DONE. This step mails people ...` | show the summary to the person you work for; call again with `confirm: true` only after they agree |
| `This API key is read-only` | the key was made in read mode; the owner can create a write key in admin `#ai` |
| `The plan of this ClickAndSay account has ended, so this API key is suspended` | every call is refused until the owner chooses a plan in admin; the same key works again afterwards |
