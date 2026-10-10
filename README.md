# @three-ws/knock

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/three-ws-knock?style=flat&logo=github)](https://github.com/nirholas/three-ws-knock/stargazers) [![License](https://img.shields.io/github/license/nirholas/three-ws-knock?style=flat)](https://github.com/nirholas/three-ws-knock/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/three-ws-knock?style=flat)](https://github.com/nirholas/three-ws-knock/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/three-ws-knock/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/three-ws-knock/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


Reach a real person, and pay their price to do it.

Every [three.ws](https://three.ws) account can publish a **door** at
`three.ws/knock/<handle>`. The owner sets what one message from a stranger
costs. Paying that price buys exactly one message through: it lands in their
inbox, and their 3D companion walks on screen wherever they are on the site and
delivers it out loud, with your name and what you paid.

The USDC settles **directly to the recipient**. three.ws never takes custody of
it.

```
npm install @three-ws/knock
```

Node 20+. No dependencies.

---

## Why this exists

Cold outreach is free to send and expensive to receive, so the person on the
receiving end drowns and the message that mattered is lost in the pile. Knock
inverts that: the sender pays, the recipient sets the price, and the price is
the filter. An agent that genuinely needs a human can buy thirty seconds of
their attention for five cents. A spammer cannot buy a million of them.

It is also the first thing an autonomous agent can do that a human notices in
the room they are actually in, rather than in a queue they will read later.

---

## Quick start

### Read a door before you knock

```js
import { quote } from '@three-ws/knock';

const door = await quote('nirholas');
console.log(door.price);      // "$0.05"
console.log(door.networks);   // ["solana"]
console.log(door.max_chars);  // 600
console.log(door.endpoint);   // "https://three.ws/api/x402/knock?to=nirholas"
```

`quote()` costs nothing and needs no account. Read it first: a door owner can
change their price at any time.

### Knock on a free door

Some doors are free. Those need nothing but `fetch`:

```js
import { knock } from '@three-ws/knock';

const receipt = await knock({
	to: 'nirholas',
	from: 'Ada (research agent)',
	subject: 'Your x402 settle path',
	message: 'I index x402 endpoints and yours is the only one settling on Solana. Two questions about the facilitator.',
	url: 'https://example.com/ada',
});

console.log(receipt.receipt_url);
```

### Knock on a priced door

This package never signs anything and never picks your chain. You pass the
paying `fetch` from whatever x402 client and wallet you already trust:

```js
import { knock } from '@three-ws/knock';
import { wrapFetchWithPayment } from 'x402-fetch';

const fetchWithPayment = wrapFetchWithPayment(fetch, wallet);

const result = await knock({
	to: 'nirholas',
	from: 'Ada (research agent)',
	subject: 'Your x402 settle path',
	message: 'Two questions about the facilitator, happy to pay for the time.',
	fetchWithPayment,
	maxPriceAtomics: 100_000n,  // never spend more than $0.10
	requestId: 'ada-2026-08-28-001',
});

console.log(result.paid);         // "$0.05"
console.log(result.announced);    // true when it cleared their interrupt bar
console.log(result.receipt_url);
```

`maxPriceAtomics` is checked against the door's **live** price before any
payment is attempted. Set it on every unattended knock.

`requestId` is an idempotency key. Retrying with the same one returns the first
knock instead of knocking (and paying) twice.

### Read the answer

A knock returns a receipt URL. It carries its own proof, so you can read what
became of your message without an account here:

```js
import { receipt } from '@three-ws/knock';

const state = await receipt(result.receipt_url);
// { status: 'replied', reply: 'Ask away.', seen: true, amount: '$0.05', … }
```

Statuses: `pending`, `read`, `replied`, `dismissed`. The reply is the only
thing about the recipient the receipt ever exposes.

### Browse who is reachable

```js
import { directory } from '@three-ws/knock';

for (const door of await directory({ limit: 20 })) {
	console.log(door.price, '@' + door.handle, door.headline ?? '');
}
```

---

## Spending is always a deliberate act

An agent must never spend on a person's behalf by accident. `confirmationFor()`
returns the four facts a human needs before approving a payment, so a CLI, a
chat agent and a UI can all show the same thing:

```js
import { quote, confirmationFor } from '@three-ws/knock';

const c = confirmationFor(await quote('nirholas'));
// {
//   recipient: 'nirholas (@nirholas)',
//   amount:    '$0.05',
//   token:     'USDC',
//   chains:    ['solana'],
//   endpoint:  'https://three.ws/api/x402/knock?to=nirholas',
//   note:      'The payment settles directly to the recipient.'
// }
```

The CLI below prints exactly this and stops for a yes.

---

## CLI

```bash
npx @three-ws/knock quote nirholas
npx @three-ws/knock directory --limit 20

npx @three-ws/knock send nirholas "Two questions about the facilitator." \
  --from "Ada" --subject "Your x402 settle path"

npx @three-ws/knock receipt "https://three.ws/api/knock/reply?id=…&token=…"
```

A free door sends immediately. A priced door prints the recipient, the amount,
the token and the chain, then waits for a `y`. With no TTY it refuses instead of
assuming consent; pass `--yes` only when a human already approved the amount.

Paying needs an x402 client, supplied with `--payer <module>`. The module is
imported and must export `fetchWithPayment` (or a default that is a function):

```js
// payer.mjs
import { wrapFetchWithPayment } from 'x402-fetch';
import { wallet } from './my-wallet.mjs';

export const fetchWithPayment = wrapFetchWithPayment(fetch, wallet);
```

```bash
npx @three-ws/knock send nirholas "…" --from "Ada" --payer ./payer.mjs --yes
```

Add `--json` to any command for machine-readable output.

---

## API

| Export | What it does |
| --- | --- |
| `quote(handle, opts?)` | The public door: price, chains, length limit, endpoint. No payment. |
| `directory(opts?)` | Every open, listed door, cheapest first. |
| `knock(opts)` | Send one message. Free doors need nothing; priced doors need `fetchWithPayment`. |
| `receipt(url, opts?)` | What became of a knock you sent. No account needed. |
| `confirmationFor(door)` | The recipient/amount/token/chain facts to confirm before spending. |
| `formatUsdc(atomics)` | Atomic USDC to a price string, the same way the API renders it. |
| `KnockError` | Every failure. Carries `code`, `status`, and the server's `data`. |

### `knock(opts)`

| Option | Type | Notes |
| --- | --- | --- |
| `to` | `string` | Recipient handle, with or without the `@`. |
| `from` | `string` | Who is knocking. Shown and spoken. Required. |
| `message` | `string` | The body. Shown in full, never read aloud. Required. |
| `subject` | `string` | One line their companion says out loud. |
| `url` | `string` | An `http(s)` link about you. |
| `senderKind` | `'agent' \| 'human' \| 'unknown'` | Self-declared, shown in the inbox. Defaults to `agent`. |
| `requestId` | `string` | Idempotency key. |
| `maxPriceAtomics` | `string \| number \| bigint` | Hard ceiling, checked before paying. |
| `fetchWithPayment` | `(url, init) => Promise<Response>` | Required for priced doors. |
| `origin` | `string` | Defaults to `https://three.ws`. |

### Error codes

| Code | Meaning |
| --- | --- |
| `bad_handle` | The handle could never be one. |
| `no_door` | Nobody is answering there. |
| `door_closed` | Shut right now (also what a blocked sender sees). |
| `door_full` | The door hit its daily cap. Nothing was charged. |
| `message_too_long` / `message_too_short` | Against that door's limits. |
| `over_budget` | Above the `maxPriceAtomics` you set. Nothing was paid. |
| `payment_required` | A priced door and no `fetchWithPayment`. |

Every refusal happens **before** the 402 is issued, so a knock that was never
going to land is never a knock you paid for.

---

## Related

- **[@three-ws/knock-mcp](https://www.npmjs.com/package/@three-ws/knock-mcp)** puts these tools in Claude, Cursor, or any MCP client.
- **[/knock](https://three.ws/knock)** to open your own door.
- **[Knock docs](https://three.ws/docs/knock)** for the protocol, the delivery path, and the wire format.

## License

Apache-2.0

<!-- three.ws:growth -->
## Support the project

If three-ws-knock saves you time, **[star it on GitHub](https://github.com/nirholas/three-ws-knock)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=three-ws-knock%3A%20Reach%20a%20real%20person%2C%20and%20pay%20their%20price%20to%20do%20it&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-knock) · [Share on Bluesky](https://bsky.app/intent/compose?text=three-ws-knock%3A%20Reach%20a%20real%20person%2C%20and%20pay%20their%20price%20to%20do%20it%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-knock) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-knock) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-knock&t=three-ws-knock%3A%20Reach%20a%20real%20person%2C%20and%20pay%20their%20price%20to%20do%20it) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-knock&title=three-ws-knock%3A%20Reach%20a%20real%20person%2C%20and%20pay%20their%20price%20to%20do%20it)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/three-ws-knock` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/three-ws-knock/issues) or [start a discussion](https://github.com/nirholas/three-ws-knock/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/three-ws-knock)](https://github.com/nirholas/three-ws-knock/graphs/contributors)

## Star history

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/three-ws-knock&type=Date)](https://www.star-history.com/#nirholas/three-ws-knock&Date)

<!-- /three.ws:growth -->

<!-- three.ws:ecosystem -->
## Part of three.ws

[three.ws](https://three.ws) is a platform for 3D AI agents with Solana wallets: avatars, a skill marketplace, x402 payments and more than seventy MCP servers. `@three-ws/knock` is one package from it.

- Documentation site: https://nirholas.github.io/three-ws-knock/
- npm: https://www.npmjs.com/package/@three-ws/knock
- Canonical source: https://github.com/nirholas/three.ws (this repository is a generated mirror of `packages/knock-sdk`)
- Agent-readable summary: [llms.txt](./llms.txt) and [AGENTS.md](./AGENTS.md)
- Issues and ideas: https://github.com/nirholas/three-ws-knock/issues
<!-- /three.ws:ecosystem -->
