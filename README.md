# RGB BTCPay Server Plugin

Accept RGB asset payments (tokens, stablecoins) in BTCPay Server.

[![BTCPay Server](https://img.shields.io/badge/BTCPay%20Server-Plugin-brightgreen)](https://btcpayserver.org)
[![.NET 8](https://img.shields.io/badge/.NET-8.0-blue)](https://dotnet.microsoft.com)

## Features

- Accept RGB20 token payments alongside Bitcoin
- Issue new RGB assets directly from BTCPay
- Automatic invoice settlement on payment confirmation
- Full UTXO management for RGB allocations
- Native rgb-lib integration (no external RGB Node required)
- Secure local PSBT signing (mnemonic never leaves .NET)

## Installation

### Via Plugin Builder (Recommended)

1. Go to your BTCPay Server **Settings** → **Plugins**
2. Search for "RGB Payments"
3. Click **Install**
4. Restart BTCPay Server

### Manual Installation

1. Download the latest release from the [Plugin Builder](https://plugin-builder.btcpayserver.org/public/plugins)
2. Extract to your BTCPay Server plugins directory
3. Restart BTCPay Server

## Configuration

### Environment Variables

```bash
# Electrum server for blockchain data
RGB_ELECTRUM_URL=ssl://electrum.blockstream.info:60002

# Directory for RGB wallet data
RGB_DATA_DIR=/data/rgb-wallets

# RGB proxy endpoint
RGB_PROXY_ENDPOINT=rpc://proxy.iriswallet.com:443/json-rpc
```

### Configuration File

Create `rgb.json` in your BTCPay Server data directory:
```json
{
  "network": "mainnet",
  "electrum_url": "ssl://electrum.blockstream.info:60002",
  "rgb_data_dir": "/data/rgb-wallets",
  "proxy_endpoint": "rpc://proxy.iriswallet.com:443/json-rpc"
}
```

### Network Defaults

| Network | Default Electrum URL |
|---------|---------------------|
| Mainnet | ssl://electrum.blockstream.info:60002 |
| Testnet | ssl://electrum.blockstream.info:60002 |
| Regtest | tcp://127.0.0.1:50001 (local electrs) |

## Quick Start

1. **Create Store** - If you don't have one already
2. **Setup RGB Wallet** - Go to Store → RGB Wallet → Setup
3. **Issue Asset** (Optional) - RGB Wallet → Issue New Asset
4. **Configure Payment** - RGB Wallet → Settings → Select asset to accept
5. **Enable Payments** - Click "Enable RGB Payments"

## Usage

### Accepting Payments

1. Create an invoice in BTCPay
2. Customer selects "RGB" payment method
3. Customer scans QR code / copies RGB invoice
4. Customer pays with RGB-compatible wallet
5. Invoice auto-settles on confirmation

### Managing UTXOs

RGB requires "colorable" UTXOs for asset operations:

1. Go to RGB Wallet → UTXOs
2. Click "Create UTXOs" if count is low
3. Wait for confirmation

### Issuing Assets

1. Go to RGB Wallet → Issue New Asset
2. Enter ticker, name, amount, precision
3. Click Issue

## Building from Source

### Prerequisites

- .NET 8.0 SDK
- BTCPay Server source (as submodule)

### Build

```bash
# Clone with submodules
git clone --recursive https://github.com/your-org/btcpay-rgb-plugin

# Build
cd BTCPayServer.Plugins.RGB
dotnet build
```

### Plugin Builder Deployment

This plugin is designed for the [BTCPay Plugin Builder](https://github.com/btcpayserver/btcpayserver-plugin-builder):

1. Fork this repository
2. Register at https://plugin-builder.btcpayserver.org
3. Add your repository
4. Plugin Builder will build and publish automatically

## Architecture

```
BTCPayServer.Plugins.RGB/
├── Controllers/          # MVC controllers
├── Data/                 # EF Core entities & migrations
├── Models/               # View models
├── PaymentHandler/       # BTCPay payment integration
├── Services/
│   ├── RgbLibService.cs       # rgb-lib-c-sharp wrapper (Lazy Loading)
│   ├── RgbLibWalletHandle.cs  # Wallet lifecycle management
│   ├── RGBWalletService.cs    # Wallet business logic
│   ├── MemoryWalletSigner.cs  # Local PSBT signing (NBitcoin)
│   ├── RgbWalletSignerProvider.cs # Signer management
│   ├── MnemonicProtectionService.cs # Mnemonic encryption
│   └── RGBInvoiceListener.cs  # Payment detection
└── Views/                # Razor views
```

### Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    rgb-lib-c-sharp (watch-only)                  │
│                                                                  │
│   Initialized with pubkey only (NO mnemonic!)                    │
│                                                                  │
│   • blind_receive()      → RGB invoice                          │
│   • send_begin()         → unsigned PSBT                        │
│   • create_utxos_begin() → unsigned PSBT                        │
│   • list_assets()        → asset list                           │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼ Unsigned PSBT
┌─────────────────────────────────────────────────────────────────┐
│                 MemoryWalletSigner (C# / .NET)                   │
│                                                                  │
│   Mnemonic stored ONLY here (encrypted at rest)                 │
│   SignPsbtAsync() → signed PSBT                                 │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼ Signed PSBT
┌─────────────────────────────────────────────────────────────────┐
│                    rgb-lib-c-sharp                               │
│                                                                  │
│   • send_end()           → broadcast RGB transfer               │
│   • create_utxos_end()   → broadcast UTXO creation              │
└─────────────────────────────────────────────────────────────────┘
```

**Key principle:** Mnemonic NEVER leaves C# code to native Rust.

## Dependencies

- **RgbLib** v0.3.0-beta.8 - Native rgb-lib bindings
- **NBitcoin** - Bitcoin primitives and PSBT signing
- **Npgsql.EntityFrameworkCore.PostgreSQL** - Database persistence

## Troubleshooting

### "InsufficientAllocationSlots"
Create more colorable UTXOs via RGB Wallet → UTXOs → Create UTXOs

### Invoice stays pending after payment
1. Check Electrum connection (Settings → Test Connection)
2. Ensure blocks are being mined (regtest)
3. Click "Refresh" on RGB Wallet page

### Plugin not loading
Check BTCPay logs: `docker logs btcpay`

### Connection errors
Verify `RGB_ELECTRUM_URL` environment variable or `rgb.json` configuration

## Platform Support

| Platform | Status |
|----------|--------|
| Linux x64 | ✅ Supported |
| macOS ARM64 (Apple Silicon) | ✅ Supported |
| macOS x64 (Intel) | ❌ Not supported (native library not included) |
| Windows | ❌ Not supported (native library not included) |

## License

MIT License - See LICENSE file

## Support

- GitHub Issues: [Create Issue](https://github.com/your-org/btcpay-rgb-plugin/issues)
- BTCPay Server Community: https://chat.btcpayserver.org
