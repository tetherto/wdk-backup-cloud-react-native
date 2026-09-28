# @tetherto/wdk-backup-cloud-react-native

Cloud backup SDK for Expo / React Native wallet apps built with WDK (Wallet Development Kit) by Tether.
Stores an encrypted master key in **Google Drive** (appDataFolder) or **iCloud** via a clean provider abstraction.

See the [general WDK documentation](https://docs.wdk.tether.io/).

---

## Installation

```bash
npm install @tetherto/wdk-backup-cloud-react-native react-native-cloud-storage
```

> **Expo note**: the bundled Google Drive and iCloud providers depend on `react-native-cloud-storage`, which requires native modules. Use a [custom dev build](https://docs.expo.dev/develop/development-builds/introduction/) — it does NOT work with Expo Go.

---

## Requirements

| Platform      | Cloud Target | Requirement                                      |
| ------------- | ------------ | ------------------------------------------------ |
| iOS / macOS   | iCloud       | iCloud Drive enabled in Settings; user signed in |
| Android / iOS | Google Drive | OAuth2 access token with `drive.appdata` scope   |

**This SDK performs NO OAuth flows.** You must supply a valid token.

---

## Quick Start

### Google Drive

```ts
import {
  CloudBackup,
  GoogleDriveProvider,
  CloudAuthError,
  CloudUnavailableError,
  CloudStorageError,
  CloudValidationError,
} from "@tetherto/wdk-backup-cloud-react-native";

// Token must be obtained by the caller (e.g., via expo-auth-session)
const provider = new GoogleDriveProvider({ accessToken: "<your_token>" });
const cloud = new CloudBackup(provider);

// Upload (metadata must include version; savedAt is auto-filled if omitted)
const result = await cloud.uploadEncryptedKey(encryptedKey, { version: 1 });

// Download — returns the full CloudEncryptionKeyFile or null
const backup = await cloud.downloadEncryptedKey(); // CloudEncryptionKeyFile | null
if (backup) {
  console.log(backup.encryptionKey, backup.version);
}

// Delete
await cloud.deleteBackup();

// Availability check
const available = await cloud.isAvailable(); // boolean

// Existence check (without downloading)
const hasBackup = await cloud.exists(); // boolean
```

### iCloud

```ts
import {
  CloudBackup,
  ICloudProvider,
} from "@tetherto/wdk-backup-cloud-react-native";

const provider = new ICloudProvider(); // default path: wallet_backup_key.json
const cloud = new CloudBackup(provider);

await cloud.uploadEncryptedKey(encryptedKey, { version: 1 });
```

---

## Error Handling

```ts
import {
  CloudValidationError,
  CloudAuthError,
  CloudUnavailableError,
  CloudStorageError,
} from "@tetherto/wdk-backup-cloud-react-native";

try {
  await cloud.uploadEncryptedKey(encryptedKey, { version: 1 });
} catch (err) {
  if (err instanceof CloudValidationError) {
    // Empty or invalid key passed by caller
  } else if (err instanceof CloudAuthError) {
    // Re-authenticate (refresh token or prompt sign-in) and retry
  } else if (err instanceof CloudUnavailableError) {
    // No network, iCloud disabled, Drive service down
  } else if (err instanceof CloudStorageError) {
    // Quota exceeded, I/O error, malformed response
  }
}
```

Each error carries a machine-readable `code` discriminant:

| Class                   | `code`                   |
| ----------------------- | ------------------------ |
| `CloudValidationError`  | `CLOUD_VALIDATION_ERROR` |
| `CloudAuthError`        | `CLOUD_AUTH_ERROR`       |
| `CloudUnavailableError` | `CLOUD_UNAVAILABLE`      |
| `CloudStorageError`     | `CLOUD_STORAGE_ERROR`    |

---

## API Reference

### `CloudBackup`

```ts
new CloudBackup(provider: CloudProvider)
```

| Method                 | Signature                                                    | Description                                                                                 |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `uploadEncryptedKey`   | `(key, metadata) => Promise<CloudEncryptionKeyFile \| null>` | Validate + upload. Returns the written payload. Throws `CloudValidationError` on empty key. |
| `downloadEncryptedKey` | `() => Promise<CloudEncryptionKeyFile \| null>`              | Download full backup payload or `null` if no backup exists.                                 |
| `deleteBackup`         | `() => Promise<void>`                                        | Delete backup (idempotent).                                                                 |
| `isAvailable`          | `() => Promise<boolean>`                                     | Lightweight probe — never throws.                                                           |
| `exists`               | `() => Promise<boolean>`                                     | Check if backup file exists without downloading — never throws.                             |

---

### `GoogleDriveProvider`

```ts
new GoogleDriveProvider(config: GoogleDriveConfig)

interface GoogleDriveConfig {
  accessToken: string;    // OAuth2 token — drive.appdata scope required
  filePath?: string;      // default: "wallet_backup_key.json"
  cloudEmail?: string;    // stored inside the backup file for traceability
}
```

- File stored in `appDataFolder` as `wallet_backup_key.json` (configurable via `filePath`)
- Uses `react-native-cloud-storage` with `CloudStorageScope.AppData`
- Verifies file existence after every write

---

### `ICloudProvider`

```ts
new ICloudProvider(config?: ICloudConfig)

interface ICloudConfig {
  filePath?: string;      // default: "wallet_backup_key.json"
  cloudEmail?: string;    // stored inside the backup file for traceability
}
```

- Uses `react-native-cloud-storage` (install it alongside this package)
- Uses `CloudStorageScope.AppData` (app-specific hidden folder)
- Checks iCloud availability before every operation
- Verifies file existence after every write
- Handles: iCloud not available, user not signed in, quota errors

---

## Implementing a Custom Provider

```ts
import type {
  CloudProvider,
  CloudEncryptionKeyFile,
} from "@tetherto/wdk-backup-cloud-react-native";

class MyCustomProvider implements CloudProvider {
  async upload(
    key: string,
    metadata: Record<string, unknown>,
  ): Promise<CloudEncryptionKeyFile | null> {
    /* ... */
  }
  async download(): Promise<CloudEncryptionKeyFile | null> {
    /* ... */
  }
  async delete(): Promise<void> {
    /* ... */
  }
  async isAvailable(): Promise<boolean> {
    /* ... */
  }
  async exists(): Promise<boolean> {
    /* ... */
  }
}

const cloud = new CloudBackup(new MyCustomProvider());
```

---

## Stored File Format

Both providers write the same JSON payload (`CloudEncryptionKeyFile`):

```json
{
  "encryptionKey": "<encrypted_wallet_master_key>",
  "savedAt": "2026-02-25T00:00:00.000Z",
  "platform": "ios",
  "version": 1,
  "cloudEmail": "user@example.com"
}
```

| Field           | Type                 | Description                                      |
| --------------- | -------------------- | ------------------------------------------------ |
| `encryptionKey` | `string`             | The encrypted wallet master key                  |
| `savedAt`       | `string`             | ISO-8601 UTC timestamp when the backup was saved |
| `platform`      | `"ios" \| "android"` | Platform that created this backup                |
| `version`       | `number`             | Backup version number                            |
| `cloudEmail`    | `string`             | Cloud user email that owns this backup           |

---

## Security Notes

- **Never logs** the encrypted key or access token
- **No AsyncStorage** — entirely in-request-lifecycle
- **No local persistence** — this SDK does not store backups outside provider calls
- **No OAuth flows** implemented — the caller owns credential management
- Error messages strip sensitive values

---

## Build

```bash
npm run build    # Outputs dist/ (CJS + ESM + .d.ts)
npm test         # Jest test suite
npm run test:coverage # Coverage report with thresholds
npm run typecheck # tsc --noEmit
```

---

## Publishing

**First publish:**

```bash
npm login
npm run build
npm publish --access public
```

**Publishing updates:**

```bash
# 1. Make your code changes
# 2. Bump version (choose one based on semver):
npm version patch   # 1.0.0 → 1.0.1 (bug fixes)
npm version minor   # 1.0.0 → 1.1.0 (new features, backward compatible)
npm version major   # 1.0.0 → 2.0.0 (breaking changes)

# 3. Build and publish
npm run build
npm publish --access public
```

Once the package reaches `1.0.0`, consumers using `^1.0.0` will receive patch/minor updates automatically on `npm install`.

---

## Architecture

```
src/
  types.ts                    # CloudProvider interface + config types
  errors.ts                   # CloudUnavailableError, CloudAuthError,
  │                           #   CloudStorageError, CloudValidationError
  cloudBackup.ts              # Public CloudBackup wrapper
  providers/
    googleDriveProvider.ts    # react-native-cloud-storage (AppData scope)
    iCloudProvider.ts         # react-native-cloud-storage
  index.ts                    # Public barrel (named exports only)
  __tests__/
    errors.test.ts
    cloudBackup.test.ts
    googleDriveProvider.test.ts
    iCloudProvider.test.ts
  __mocks__/
    react-native-cloud-storage.ts
examples/
  usage.ts
```

---

## Integration with `@tetherto/wdk-backup-remote`

```ts
import { BackendBackupClient } from "@tetherto/wdk-backup-remote";
import type { SeedItem, EntropyItem } from "@tetherto/wdk-backup-remote";
import {
  CloudBackup,
  GoogleDriveProvider,
} from "@tetherto/wdk-backup-cloud-react-native";

const backend = new BackendBackupClient({
  baseUrl: "https://api.mywallet.com",
});
const cloud = new CloudBackup(new GoogleDriveProvider({ accessToken }));

// Upload (seed, entropy, and cloud key)
await backend.uploadSeed({
  seed: encryptedSeed,
  authToken,
  metadata: { device: "ios" },
});
await backend.uploadEntropy({ entropy: encryptedEntropy, authToken });
await cloud.uploadEncryptedKey(encryptedMasterKey, { version: 1 });

// Retrieve — backend returns full arrays of SeedItem[] / EntropyItem[]
const seeds: SeedItem[] = await backend.getSeed(authToken); // [] if no backup
const entropies: EntropyItem[] = await backend.getEntropy(authToken); // [] if no backup
const backup = await cloud.downloadEncryptedKey(); // CloudEncryptionKeyFile | null
```

---

## License

Apache-2.0
