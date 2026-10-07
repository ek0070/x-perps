# Tweet Terminal (perps)

A perpetual-futures trading terminal that renders inside X posts through X's Player Card.
Market data and order execution are Hyperliquid; wallets are Privy embedded wallets (non-custodial).

## Routes

| Route | What it is |
| --- | --- |
| `/` and `/pulse` | The terminal: Movers / Volume / Funding columns, live. Tabs on phones. |
| `/t/<COIN>` | The link you post. Player-card + Open Graph tags in `<head>`, full terminal in the body. |
| `/embed/<COIN>` | The 480x480 player X frames: price, chart, wallet, Long / Short, close. |
| `/t/pulse`, `/embed/pulse` | Share link and player for the whole live feed. |
| `/about`, `/terms` | Landing page with the link maker; terms. |
| `/api/markets`, `/api/markets/stream` | All markets as JSON; the same as server-sent events (`?coin=BTC` for one). |
| `/api/candles/<COIN>`, `/api/trades/<COIN>` | Chart candles and the recent-trades tape. |
| `/api/preview/<COIN>`, `/api/preview/pulse` | Generated 1200x630 card images. |

`<COIN>` is a Hyperliquid perp ticker (`BTC`, `ETH`, `SOL`, `HYPE`, `kPEPE`, ...). Unknown tickers return 404.

## Environment variables

| Variable | Required | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_SITE_URL` | yes | Public origin, no trailing slash. Goes into the card tags and share links. |
| `NEXT_PUBLIC_PRIVY_APP_ID` | yes, to trade | Without it the site shows data but nobody can log in. |
| `NEXT_PUBLIC_X_HANDLE` | no | Default `@tweetterminal`. |
| `NEXT_PUBLIC_SITE_NAME` | no | Default `Tweet Terminal`. |
| `NEXT_PUBLIC_ARBITRUM_RPC_URL` | no | RPC the deposit dialog reads balances from. Default is the public Arbitrum endpoint. |

There are no server secrets: Hyperliquid's API needs no key. `NEXT_PUBLIC_*` values are compiled into the
build, so changing one requires a redeploy.

## Privy dashboard (required before anyone can log in)

1. **Allowed origins**: add your site origin (and `http://localhost:3150` for local testing).
2. **Login methods**: enable Email and Twitter/X.
3. **Embedded wallets**: enable **Ethereum** wallets, created on login. Hyperliquid accounts are EVM addresses.

## How trading works

- Login creates an embedded EVM wallet. Its address is the user's Hyperliquid account.
- Deposit: send USDC on Arbitrum to that address, then "Move USDC to trading account" sends it to the Hyperliquid
  bridge (minimum 5 USDC, needs a little ETH on Arbitrum for gas). Or send USDC to the address from an existing
  Hyperliquid account, which is instant and free.
- Long / Short: sets leverage, then places an immediate-or-cancel order priced at mark +/- slippage
  (default 1%, changeable in Settings). Margin presets $10 / $50 / $100 or custom; minimum position is $10.
- Close 25% / 50% / 100%: a reduce-only order against the open position.
- Orders are signed in the browser by the user's wallet and sent straight to Hyperliquid. The server never sees a key.

`node scripts/verify-signing.mjs` proves the order path without funds: it signs an order with a throwaway key and
checks that the exchange recovers that key's address from the signature.

## Data and live updates

One server-side poller reads every market from Hyperliquid every 3 seconds while anyone is connected and fans the
result out over server-sent events, so upstream cost does not grow with viewers. Candles are cached 5 s and trades
3 s. Browsers fall back to polling if the stream drops. Account balances and positions are read by the browser
directly from Hyperliquid.

## Run locally

```
npm install --legacy-peer-deps
npm run build
npx next start -p 3150
```

## Deploy

Railway (from the project folder, after `git init` and pushing to GitHub):

```
railway init --name tweet-terminal-perps
railway add --service web
railway domain                       # prints your https://...up.railway.app domain
railway variables --service web --set NEXT_PUBLIC_SITE_URL=https://<that-domain> --set NEXT_PUBLIC_PRIVY_APP_ID=<id>
railway up --service web             # or connect the GitHub repo for auto-deploys
```

Vercel: import the repo, set the same variables, deploy. Note that the shared poller and in-memory cache work best
on a long-running server (Railway); on serverless each instance keeps its own.

Custom domain: add it in the host's dashboard, create the CNAME it shows at your registrar, then change
`NEXT_PUBLIC_SITE_URL` to the new origin and redeploy so the card URLs use it. Add the new origin in Privy too.

## Testing the card

Paste `https://<domain>/t/BTC` into the X composer. If X shows a plain summary instead of the player, that is X's
Player Card approval for the domain, not the site.
