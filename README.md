# SecureAuth - TOTP and Two-Factor Authentication Suite

SecureAuth is a production-grade, client-side, zero-knowledge Time-Based One-Time Password (TOTP) authenticator and security demonstration platform. It implements RFC 6238 and RFC 4226 specifications natively in the browser using the W3C Web Crypto API. The application includes a multi-account authenticator vault, real-time camera QR scanner, step-by-step cryptographic visualizer, client-side AES-GCM encrypted backup engine, live 2FA verification simulator, and Progressive Web App (PWA) offline capability.

## Tech Stack

### Core Framework and Language
- React 19: Component-based UI architecture with functional components and hooks.
- TypeScript 7: Strict static typing across all cryptographic models, account schemas, and application state.
- Vite 8: High-performance frontend build tool and local development server.

### Styling and User Interface
- Tailwind CSS v4: Modern utility-first CSS engine integrated via `@tailwindcss/vite`.
- Lucide React: Vector iconography for navigation, security status indicators, and account management.
- Motion: Transition animations for countdown progress rings, modal dialogues, and view switching.

### Cryptography and Security
- W3C Web Crypto API (`window.crypto.subtle`):
  - HMAC generation: Native hardware-accelerated HMAC-SHA1, HMAC-SHA256, and HMAC-SHA512.
  - Key derivation: PBKDF2 with SHA-256 and 100,000 iterations for master password hashing and vault keys.
  - Symmetric encryption: AES-GCM with 256-bit key length, cryptographically secure 16-byte salt, and 12-byte initialization vectors (IV).
  - Randomness: Cryptographically secure pseudo-random number generator (`crypto.getRandomValues`).

### Barcode and QR Utilities
- jsQR: Real-time QR code reading from video canvas streams for camera-based account import.
- qrcode: Dynamic generation of high-contrast QR codes and SVG vectors for account sharing and pairing.

### Web Platform and PWA
- vite-plugin-pwa: Service worker registration, asset pre-caching, and Web App Manifest generation.
- WebAPK Compliance: Full compatibility with Android WebAPK minting and standalone desktop PWA installation.

## Architecture and Project Structure

```
├── public/
│   ├── favicon.svg              # Application vector favicon
│   ├── manifest.json            # PWA manifest metadata
│   ├── pwa-192x192.png          # Standard PWA icon asset
│   ├── pwa-512x512.png          # High-resolution PWA splash asset
│   └── downloads/               # Offline distribution artifacts
├── src/
│   ├── components/
│   │   ├── AccountQrModal.tsx     # QR code export modal for cross-device migration
│   │   ├── AddAccountModal.tsx    # Manual key entry and URI parser modal
│   │   ├── AuthenticatorView.tsx  # Primary dashboard with accounts list, timers, and tools
│   │   ├── DemoSaaSView.tsx       # Live 2FA login verification simulation playground
│   │   ├── GitHubHelperModal.tsx  # GitHub two-factor authentication setup assistant
│   │   ├── InstallApkModal.tsx    # Android WebAPK and installation instruction modal
│   │   ├── LockScreen.tsx         # Master password / PIN vault unlock interface
│   │   ├── OtpCard.tsx            # Account card with countdown circle and copy actions
│   │   ├── QrScannerModal.tsx     # HTML5 camera feed and live video frame analyzer
│   │   ├── SecurityModal.tsx      # Vault export/import with AES-GCM passphrase encryption
│   │   └── TotpVisualizerView.tsx # Interactive step-by-step cryptographic decomposition
│   ├── data/
│   │   └── defaultAccounts.ts     # Pre-configured test accounts and security presets
│   ├── hooks/
│   │   └── useTotpTicker.ts       # Centralized 1-second interval ticker for synchronized timers
│   ├── types/
│   │   └── auth.ts                # TypeScript interfaces for accounts, algorithms, and results
│   ├── utils/
│   │   ├── brandIcons.ts          # Brand color matching and SVG icons for popular providers
│   │   └── totp.ts                # Pure RFC 6238 / RFC 4226 / RFC 4648 cryptographic algorithms
│   ├── App.tsx                    # Root application state, view routing, and vault management
│   ├── index.css                  # Global Tailwind CSS directives and theme variables
│   └── main.tsx                   # React root mount point
├── package.json                   # Dependency definitions and lifecycle scripts
├── tsconfig.json                  # TypeScript compiler configuration
└── vite.config.ts                 # Vite bundler, PWA, and Tailwind plugin configuration
```

## How It Works

### 1. RFC 6238 TOTP Algorithm Implementation

The core algorithm produces a time-synchronized, one-time passcode from a shared Base32 secret string. It executes the following mathematical sequence:

#### Step 1: Base32 Decoding (RFC 4648)
The Base32 secret (characters `A-Z` and `2-7`) is stripped of whitespace, dashes, and padding, then converted into raw binary bytes. Every 8 characters represent 5 bytes (40 bits).

#### Step 2: Time Step Calculation
The current Unix timestamp in seconds is divided by the time period (standard is 30 seconds):
```
TimeStep = floor(CurrentUnixTimestamp / Period)
```
The remaining seconds in the current cycle is calculated as:
```
SecondsRemaining = Period - (CurrentUnixTimestamp % Period)
```

#### Step 3: Counter Byte Serialization
The integer `TimeStep` is converted into an 8-byte (64-bit) big-endian array.

#### Step 4: HMAC Calculation
Using the W3C Web Crypto API, the decoded secret bytes are imported as an HMAC signing key with the specified hash function (`SHA-1`, `SHA-256`, or `SHA-512`). The key signs the 8-byte counter buffer to produce the HMAC digest:
```
HMAC = HMAC-SHA(SecretBytes, CounterBytes)
```

#### Step 5: Dynamic Truncation (RFC 4226 Section 5.3)
To extract a deterministic 4-byte segment from the digest:
- The low 4 bits of the last byte in the HMAC digest define the offset index:
  ```
  Offset = HMAC[HMAC.length - 1] & 0x0F
  ```
- Four consecutive bytes starting at `Offset` are extracted.
- The most significant bit of the first byte is masked with `0x7F` to guarantee a positive 31-bit integer, preventing sign-extension errors:
  ```
  BinaryCode = ((HMAC[Offset] & 0x7F) << 24)
             | ((HMAC[Offset + 1] & 0xFF) << 16)
             | ((HMAC[Offset + 2] & 0xFF) << 8)
             | (HMAC[Offset + 3] & 0xFF)
  ```

#### Step 6: Modulo Reduction and Formatting
The resulting 31-bit integer is reduced modulo 10 raised to the target number of digits (typically 6 or 8):
```
OTP = BinaryCode % (10 ^ Digits)
```
The number is left-padded with zeros to match the specified digit length.

### 2. Time Drift and Synchronization Window

Real-world servers and devices may experience clock skew. The verification engine (`verifyTotp`) validates entered codes against a configurable drift window:
- Window = 1 evaluates the current time step (`T`), the previous time step (`T - 1`), and the next time step (`T + 1`).
- This grants a 90-second tolerance window (30 seconds before to 30 seconds after) without weakening security parameters.

### 3. Client-Side Zero-Knowledge Vault Encryption

All authenticator secrets remain solely within the user's browser. No secrets, keys, or logs are transmitted over the network.

When the user exports or protects their vault with a password:
1. A cryptographically secure 16-byte salt and 12-byte initialization vector (IV) are generated via `crypto.getRandomValues`.
2. The user's passphrase is fed into PBKDF2 with SHA-256 using 100,000 iterations to derive a 256-bit AES key.
3. The vault accounts JSON payload is encrypted using AES-GCM with the derived key and random IV.
4. The output format contains the salt, IV, and ciphertext encoded in hexadecimal strings for portable backup files.

### 4. Interactive Cryptographic Visualizer

The application includes an educational visualizer (`TotpVisualizerView`) that decomposes the mathematical operations step by step in real time:
- Displays current Unix epoch seconds and the active 30-second block index.
- Shows the Base32 decoded secret in hexadecimal notation.
- Renders the 8-byte big-endian counter representation.
- Displays the complete HMAC hash output and highlights the 4-bit offset selector.
- Visualizes the 4-byte slice extraction, masking operation, and final modulo math.

### 5. Live QR Code Scanner

The camera scanner (`QrScannerModal`) leverages the HTML5 `MediaDevices.getUserMedia` API:
- Streams video directly into an offscreen HTML5 canvas element at 30 frames per second.
- Analyzes imageData frame buffers using `jsQR`.
- Parses standard `otpauth://totp/Label?secret=KEY&issuer=Issuer` URI schemes.
- Automatically handles URL-encoded labels, custom digit counts (6 or 8), custom intervals, and algorithm flags (`SHA1`, `SHA256`, `SHA512`).

### 6. Progressive Web App and Android WebAPK

SecureAuth functions entirely offline once loaded:
- Service Worker caches core HTML, JavaScript bundles, CSS, and icon assets.
- Responsive design tailored for mobile viewport dimensions and desktop browsers.
- Android devices can install the app directly via Google Chrome, which mints a native Android package (WebAPK) on the device, providing an app icon on the home screen and isolated sandbox execution without browser address bars.

## How It Was Created

1. Zero External Cryptographic Dependencies:
   Instead of bundling legacy JavaScript crypto libraries, SecureAuth relies exclusively on the browser's native `SubtleCrypto` interface. This ensures constant-time execution paths where possible, hardware optimization, and zero risk of supply-chain vulnerabilities in third-party crypto packages.

2. State Management and Synchronization:
   A dedicated ticker hook (`useTotpTicker`) drives a single, synchronized requestAnimationFrame/interval loop. All active account cards read from this single source of truth, guaranteeing that countdown rings and TOTP code updates across the entire application remain in exact lockstep.

3. Strict Type Safety:
   Every parameter conforming to RFC 6238 (algorithms, secret formats, period parameters, and error types) is modeled strictly with TypeScript interfaces, eliminating runtime type mismatches.

4. Defense in Depth:
   The application incorporates automatic clipboard clearing timers, masked secret fields, session lock timeouts, and tamper-resistant backup verification routines.

## Development and Build Scripts

### Prerequisites
- Node.js version 18.0.0 or higher
- npm version 9.0.0 or higher

### Installation
Clone the repository and install all project dependencies:
```bash
npm install
```

### Run Local Development Server
Start the Vite development server on port 3000:
```bash
npm run dev
```

### Static Type Check
Validate the entire codebase for TypeScript compiler errors:
```bash
npm run lint
```

### Production Build
Compile TypeScript, bundle assets, and generate the production distribution:
```bash
npm run build
```
The compiled output will be generated in the `dist` directory.

### Preview Production Build
Serve the production bundle locally for testing:
```bash
npm run preview
```

## Standards and RFC Compliance

- RFC 6238: TOTP - Time-Based One-Time Password Algorithm
- RFC 4226: HOTP - An HMAC-Based One-Time Password Algorithm
- RFC 4648: The Base16, Base32, and Base64 Data Encodings
- NIST SP 800-63B: Digital Identity Guidelines - Authentication and Lifecycle Management

## License

This project is licensed under the MIT License.
