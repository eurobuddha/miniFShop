# miniFShop

A lightweight MiniDapp shop template for selling physical or digital products on the Minima blockchain. Customers pay in MINI or USDT. Order details are encrypted on-chain — only the vendor can read them.

Includes a **customer-facing shop**, a **vendor inbox (mInbox)**, and a **Studio desktop app** for building shops without touching code.

---

## Contents

- [How It Works](#how-it-works)
- [Getting Started](#getting-started)
  - [Option A: Studio (Desktop App)](#option-a-studio-desktop-app)
  - [Option B: CLI Build](#option-b-cli-build)
- [Deploy Your Shop](#deploy-your-shop)
- [mInbox: Managing Orders](#minbox-managing-orders)
- [ChainMail: Replying to Buyers](#chainmail-replying-to-buyers)
- [Currency Options](#currency-options)
- [Configuration Reference](#configuration-reference)
- [Building the Studio Installer](#building-the-studio-installer)
- [File Structure](#file-structure)
- [Security Model](#security-model)
- [Requirements](#requirements)
- [License](#license)

---

## How It Works

miniFShop produces two MiniDapps:

| MiniDapp | Who uses it | What it does |
|---|---|---|
| **Shop** (`[productname].mds.zip`) | Customers | Displays product, collects order, sends payment |
| **mInbox** (`mInbox.zip`) | Vendor | Receives orders, manages status, replies to buyers |

### Order Flow

```
Customer                          Minima Blockchain              Vendor (mInbox)
   │                                     │                            │
   │  Fills email, address, message      │                            │
   │  ──────────────────────────────►    │                            │
   │                                     │                            │
   │  Order encrypted with vendor pubkey │                            │
   │  Payment + ciphertext sent on-chain │                            │
   │  ──────────────────────────────►    │                            │
   │                                     │  New coin at vendor addr   │
   │                                     │  ────────────────────────► │
   │                                     │                            │  Decrypts with own key
   │                                     │                            │  Reads: email, address,
   │                                     │                            │  message, buyer MX key
   │  TX ID shown as confirmation        │                            │
```

1. Customer selects quantity and fills in email, shipping address, and an optional message.
2. The order payload is encrypted with the **vendor's Maxima public key** using ChainMail (`maxmessage action:encrypt`). Only the vendor can decrypt it.
3. A single Minima transaction sends the payment **and** attaches the encrypted order in state port 99.
4. The vendor's mInbox scans for incoming coins, decrypts each order, and stores it in a local SQL database.
5. The vendor can update order status (PAID → PREPARING → SHIPPED) and reply directly via ChainMail.

Nothing readable is stored on-chain. The buyer's contact details and public key are inside the ciphertext.

---

## Getting Started

### Prerequisites

- A running Minima node with MDS (MiniDapp System) enabled
- Your **Minima wallet address** (`0x...`) — visible in your wallet
- Your **Maxima public key** (`Mx...`) — run `maxima action:info` in the MDS terminal

### Option A: Studio (Desktop App)

Studio is a local web UI that builds shops without any command-line work. Download the installer for your platform from the [Releases](https://github.com/eurobuddha/miniFShop/releases) page.

**macOS**: Open the `.dmg`, drag the app to Applications, double-click to launch.
**Windows**: Run the `-Setup.exe` installer (no admin required), launch from the Start Menu or desktop shortcut.

Studio opens automatically in your browser at `http://localhost:3456`.

#### Steps

1. **Vendor Setup tab** — enter your wallet address and Maxima public key, click Save.
2. **Build Shop tab** — fill in:
   - Product name
   - Description
   - Price per unit (in MINI or USDT)
   - Maximum units per order
   - Product image (SVG recommended; PNG/JPG accepted but keep under 10 KB)
   - Currency (MINI or USDT)
3. Click **Build Shop**.
4. Download:
   - `[productname].mds.zip` — your shop MiniDapp
   - `mInbox.zip` — your vendor inbox MiniDapp

---

### Option B: CLI Build

If you prefer the terminal:

```bash
git clone https://github.com/eurobuddha/miniFShop.git
cd miniFShop
npm install
```

```bash
node build-miniFShop.js <address> <pubkey> [options]
```

| Option | Description | Default |
|---|---|---|
| `<address>` | Your Minima wallet address (`0x...`) | required |
| `<pubkey>` | Your Maxima public key (`Mx...`) | required |
| `--name <name>` | Product name | `"Product"` |
| `--description <desc>` | Product description | — |
| `--price <price>` | Price per unit | `1` |
| `--max-units <max>` | Maximum units per order | `10` |
| `--image <path>` | Path to product image (SVG/PNG/JPG) | default SVG |

**Example:**

```bash
node build-miniFShop.js \
  0xA65ED661F4B6580AC93BEA7E07A36D98CF3EA0E2F3B5D1E7A92C4B6F0D3E5A1 \
  MxG18HGG4D3F2A1B9C7E6D5F4A3B2C1D0E9F8A7B6C5D4E3F2A1B0C9D8E7F6A5B4 \
  --name "Organic Honey" \
  --description "Pure raw honey from local bees" \
  --price 2.5 \
  --max-units 5 \
  --image ./honey.svg
```

Output:

```
dist/
├── organic-honey.mds.zip   # Shop MiniDapp
└── mInbox.zip              # Vendor inbox
```

---

## Deploy Your Shop

### 1. Install mInbox on your Minima node

1. Open your **MDS Hub** (`http://localhost:9003` or your node's MDS port).
2. Click **Install MiniDapp**.
3. Upload `dist/mInbox.zip`.
4. mInbox appears in your MDS app list. Open it — it starts scanning for orders immediately.

### 2. Publish your shop to MiniFS

MiniFS is Minima's on-chain file system. Files published there are accessible to any Minima user.

1. Open the **MiniFS** MiniDapp in your MDS Hub (install it if you haven't).
2. Upload `dist/[productname].mds.zip`.
3. Copy the MiniFS link (format: `mxs://...` or `http://...`).
4. Share the link with customers. They open it in their own MDS Hub and install the shop MiniDapp.

> **Size limit**: Shop packages must be under **50 KB**. Use SVG images to stay well within this limit. The builder warns you if you're close to the limit.

---

## mInbox: Managing Orders

mInbox is the vendor's order management panel. It runs on your Minima node and automatically decrypts incoming orders.

### Features

- **Order list** with status badges (PAID / PREPARING / SHIPPED)
- **Unread indicator** — new orders are highlighted until opened
- **Filter by status** — view all orders or filter by current state
- **Order detail modal** — shows product, amount, currency, email, shipping address, optional message, timestamp, and TX ID
- **Status updates** — advance orders through PAID → PREPARING → SHIPPED
- **ChainMail button** — open ChainMail with the buyer's address ready to use

### How Orders Arrive

mInbox scans the blockchain for coins arriving at your vendor address. For each incoming coin:

1. It reads state port 99 (the encrypted order payload).
2. It calls `maxmessage action:decrypt` — this only succeeds on your node, using your private key.
3. If decryption is valid, the order is saved to the local SQL database.
4. The order appears in the list immediately.

Orders are deduplicated by coin transaction ID, so restarting mInbox or rescanning never creates duplicates.

### Database

Orders are stored in an H2 SQL database managed by your Minima node:

| Column | Description |
|---|---|
| `ref` | Order reference ID |
| `product` | Product name |
| `amount` | Quantity ordered |
| `currency` | MINI or USDT |
| `email` | Buyer's email address |
| `shipping` | Buyer's shipping address |
| `message` | Optional message from buyer |
| `timestamp` | Order time (Unix ms) |
| `coinid` | Transaction ID (unique) |
| `isread` | Read/unread flag |
| `buyerPublicKey` | Buyer's Maxima public key (for ChainMail) |
| `status` | PAID / PREPARING / SHIPPED |

---

## ChainMail: Replying to Buyers

When a buyer places an order, their **Maxima public key** is included inside the encrypted order payload. After you decrypt the order, mInbox extracts this key and stores it.

To send an encrypted reply to a buyer:

1. Open the order in mInbox.
2. Click **Open ChainMail** — the buyer's Maxima public key is copied to your clipboard and ChainMail opens in a new tab.
3. Paste the key into ChainMail's recipient field and write your message.

All ChainMail messages are end-to-end encrypted. The buyer receives your reply in their own ChainMail inbox.

---

## Currency Options

miniFShop supports two currencies:

| Currency | Token ID | Notes |
|---|---|---|
| **MINI** (Minima) | `0x00` | Native Minima token |
| **USDT** (Tether) | `0x7D39...` | Bridged USDT on Minima |

Select the currency when building your shop. The shop UI displays the correct icon and label automatically. The token ID is hardcoded into the shop payment transaction.

---

## Configuration Reference

### `miniFShop-shop/config.js`

Generated by the builder. Injected into the shop MiniDapp.

```js
const VENDOR_CONFIG = {
    vendorAddress: "0x...",       // Payment destination address
    vendorPublicKey: "Mx...",     // Vendor's Maxima public key (for encryption)
    appName: "Your Product",
    version: "1.0.0"
};
```

### `miniFShop-shop/dapp.conf`

MDS manifest. Controls how the shop appears in the MDS Hub.

```json
{
    "name": "Your Product",
    "icon": "icon.svg",
    "version": "1.0.0",
    "description": "Buy Your Product with Minima",
    "category": "Commerce"
}
```

### `mInbox/config.js`

Generated by the builder. Injected into mInbox.

```js
const VENDOR_ADDRESS = "0x...";
const VENDOR_PUBLIC_KEY = "Mx...";
```

### Template Placeholders (`index.template.html`)

The shop HTML is generated from a template. These placeholders are replaced at build time:

| Placeholder | Value |
|---|---|
| `{{PRODUCT_NAME}}` | Product title |
| `{{PRODUCT_DESCRIPTION}}` | Product description text |
| `{{PRODUCT_PRICE}}` | Price per unit |
| `{{MAX_UNITS}}` | Maximum order quantity |
| `{{PRODUCT_IMAGE}}` | Image filename (`product.svg`, `product.png`, etc.) |
| `{{CURRENCY_LABEL}}` | `"Minima"` or `"USDT"` |
| `{{TOKEN_ID}}` | Blockchain token ID |
| `{{CURRENCY_ICON}}` | Icon file (`minima_logo_bw.svg` or `usdt_icon.svg`) |
| `{{VENDOR_ADDRESS}}` | Vendor wallet address |
| `{{VENDOR_PUBLIC_KEY}}` | Vendor Maxima public key |

---

## Building the Studio Installer

The Studio desktop app is a compiled Node.js binary served via a local HTTP server. The build scripts produce platform-specific installers.

### macOS

Requirements: Node.js, macOS with Xcode command-line tools.

```bash
npm run build:mac
```

Output: `release/miniFShop-Studio-1.0.0.dmg`

What it does:
1. Compiles `studio.js` + `studio-builder.js` to a standalone macOS arm64 binary using `@yao-pkg/pkg`.
2. Wraps it in a `.app` bundle with `Info.plist` metadata and `icon.icns`.
3. Ad-hoc code-signs the bundle and removes the quarantine flag.
4. Packages everything into a `.dmg` disk image.

Target: macOS 12+ (Apple Silicon; Intel Macs supported via Rosetta 2).

### Windows

Requirements: `makensis` (NSIS), ImageMagick (`convert`). Runs on macOS cross-compiling for Windows.

```bash
npm run build:win
```

Output: `release/miniFShop-Studio-1.0.0-Setup.exe`

What it does:
1. Downloads portable Node.js v22.14.0 for Windows (cached in `build/.node-win-cache/`).
2. Stages all app files and `node_modules` to `release/staging/`.
3. Generates `icon.ico` from `icon.svg` using ImageMagick.
4. Builds an NSIS installer with per-user install (no UAC/admin required).
5. Installs to `%LOCALAPPDATA%\miniFShop Studio` with a desktop shortcut and Start Menu entry.
6. Uses `launch.vbs` to start the server silently in the background.

### Running Studio without an installer

```bash
npm run studio
# Opens http://localhost:3456 in your browser
```

---

## File Structure

```
miniFShop/
│
├── studio.js                   # Studio server (Node.js, port 3456)
├── studio-builder.js           # Shop/inbox zip generator
├── build-miniFShop.js          # CLI build entry point
│
├── web/                        # Studio web UI
│   ├── index.html              # Two-tab UI: Vendor Setup + Build Shop
│   ├── app.js                  # Frontend logic (config save, image upload, build)
│   └── style.css
│
├── miniFShop-shop/             # Shop MiniDapp source
│   ├── index.template.html     # HTML template with {{placeholders}}
│   ├── index.html              # Generated output (overwritten on build)
│   ├── config.js               # Vendor credentials (generated)
│   ├── dapp.conf               # MDS manifest (generated)
│   ├── style.css               # Shop UI styles
│   ├── mds.js                  # Minima JS library
│   ├── icon.svg                # App icon
│   ├── product.svg             # Default product image
│   ├── minima_logo_bw.svg      # MINI currency icon
│   └── usdt_icon.svg           # USDT currency icon
│
├── mInbox/                     # Vendor inbox MiniDapp source
│   ├── index.html              # Inbox UI
│   ├── app.js                  # Order scan, decrypt, store, display
│   ├── config.js               # Vendor credentials (generated)
│   ├── dapp.conf               # MDS manifest
│   ├── style.css               # Inbox UI styles
│   ├── mds.js                  # Minima JS library
│   └── icon.svg                # App icon
│
├── build/                      # Installer build scripts
│   ├── build-mac.sh            # macOS .app + .dmg builder
│   ├── build-win.sh            # Windows NSIS installer builder
│   ├── installer.nsi           # NSIS installer script
│   ├── Info.plist              # macOS app metadata
│   ├── icon.icns               # macOS app icon
│   └── icon.ico                # Windows app icon (generated)
│
├── dist/                       # Build output (MiniDapp packages)
│   ├── [productname].mds.zip   # Shop MiniDapp
│   └── mInbox.zip              # Vendor inbox
│
├── release/                    # Installer output
│   ├── miniFShop-Studio-1.0.0.dmg
│   └── miniFShop-Studio-1.0.0-Setup.exe
│
└── package.json
```

---

## Security Model

### Encryption

Order payloads are encrypted using Minima's **ChainMail** protocol (`maxmessage action:encrypt`):

- The **vendor's Maxima public key** is used for encryption. Only the vendor's Minima node, holding the corresponding private key, can decrypt.
- On-chain, state port 99 contains only the raw ciphertext. No buyer details, no buyer identity, no public key are readable from the chain data.
- The buyer's Maxima public key is embedded inside the encrypted payload (`buyerMxKey` field) — it is recovered after decryption and stored locally in the vendor's database for ChainMail replies.

### What is visible on-chain

| Data | On-chain visibility |
|---|---|
| Payment amount | Public |
| Vendor address | Public |
| Buyer address | Public (from the sending coin) |
| Order ciphertext (state 99) | Public but unreadable without vendor's private key |
| Email, shipping, message | Hidden (inside ciphertext) |
| Buyer's Maxima public key | Hidden (inside ciphertext) |

### SQL injection and XSS

mInbox uses `escapeSQL()` on all user-supplied values before database writes and `escapeHtml()` before rendering any order data into the UI.

---

## Requirements

- **Minima node** with MDS enabled (tested on Minima 1.x)
- **Node.js ≥ 16** (for CLI and Studio builds)
- Shop packages must be **under 50 KB** (MiniFS limit)
- Product images: **SVG preferred** (smallest size, scales perfectly). PNG/JPG accepted — keep under 10 KB to stay within the size limit.
- For Windows installer builds: `makensis` and `convert` (ImageMagick) must be on your PATH

---

## License

MIT
