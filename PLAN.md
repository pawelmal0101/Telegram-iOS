# Telegram iOS Fork — Customization Plan

## App Configuration

| Key | Value |
|-----|-------|
| api_id | `25514754` |
| api_hash | `c103d63145ec214a9e0ce0a6419df11c` |
| App title | newsbotaggregator |
| Short name | newsbotaggregator |

---

## Step 0: Prerequisites

- [ ] Xcode 15+ installed
- [ ] Python 3 installed
- [ ] Bazel (installed automatically by build system)
- [ ] Apple Developer account (free works for device testing, paid for distribution)
- [ ] Your iPhone connected + trusted

---

## Step 1: Create Build Configuration

Create `build-system/my-configuration.json`:

```json
{
  "bundle_id": "org.newsbotaggregator.Telegram",
  "api_id": "25514754",
  "api_hash": "c103d63145ec214a9e0ce0a6419df11c",
  "team_id": "5M33F9J358",
  "app_center_id": "0",
  "is_internal_build": "true",
  "is_appstore_build": "false",
  "appstore_id": "0",
  "app_specific_url_scheme": "tgnewsbot",
  "premium_iap_product_id": "",
  "enable_siri": false,
  "enable_icloud": false
}
```

Find your Team ID: Xcode > Settings > Accounts > your Apple ID > Team ID column.

---

## Step 2: Generate Xcode Project

```sh
python3 build-system/Make/Make.py \
  --cacheDir="$HOME/telegram-bazel-cache" \
  generateProject \
  --configurationPath=build-system/my-configuration.json \
  --xcodeManagedCodesigning
```

This generates an `.xcworkspace` you can open in Xcode. Xcode-managed codesigning handles provisioning profiles automatically (no git codesigning repo needed for device builds).

---

## Step 3: Implement Mod 1 — Restrict Global Search

**Goal:** Remove global search from chat list. Keep in-chat search working.

### Files to modify

**`submodules/ChatListUI/Sources/ChatListController.swift`**
- `ChatListControllerImpl.activateSearch(...)` — main entry point for search activation
- `tabBarActivateSearch()` — tab bar search trigger
- Short-circuit these methods to `return` early (or gate behind a `UserDefaults` flag)

**`submodules/TelegramUI/Components/ChatListHeaderComponent/Sources/ChatListNavigationBar.swift`**
- `ChatListNavigationBar` — has `search: Search?` property
- Set `search` to `nil` or `isEnabled: false` to hide search bar in nav

**`submodules/ChatListUI/Sources/ChatListControllerNode.swift`**
- `activateSearch(...)` async method — also short-circuit here as backup

### What to keep
- `ChatSearchController` (in-chat search within opened conversation) — do NOT touch
- `SearchDisplayController` when opened from within a chat — do NOT touch

### Implementation approach
```swift
// In ChatListControllerImpl.activateSearch(...)
// Add at top of method:
let globalSearchEnabled = UserDefaults.standard.bool(forKey: "globalSearchEnabled") // defaults to false
guard globalSearchEnabled else { return }
```

Same guard in `tabBarActivateSearch()`.

For nav bar, in `ChatListNavigationBar` update logic, override `search` to `nil` when flag is off.

---

## Step 4: Implement Mod 2 — Filter Messages from @imaginati8n

**Goal:** Hide all messages from username `imaginati8n` in all chat views. Messages stay on server, just not rendered.

### Files to modify

**`submodules/TelegramUI/Sources/ChatHistoryEntriesForView.swift`**
- `chatHistoryEntriesForView(...)` function (line 36) — this is where all messages are processed before display
- After entries are built, filter out messages where author's `addressName` matches blocked username
- Message author peer is accessible via `message.author` → cast to `TelegramUser` → check `username` / `addressName`

```swift
// After entries array is populated, before return:
let blockedUsernames: Set<String> = ["imaginati8n"]
entries = entries.filter { entry in
    switch entry {
    case let .MessageEntry(message, _, _, _, _, _, _, _, _, _, _, _, _, _, _, _):
        if let author = message.message.author as? TelegramUser,
           let username = author.addressName,
           blockedUsernames.contains(username.lowercased()) {
            return false
        }
        return true
    case let .MessageGroupEntry(group, _):
        // Filter if ALL messages in group are from blocked user
        let allBlocked = group.allSatisfy { entry in
            if let author = entry.0.author as? TelegramUser,
               let username = author.addressName,
               blockedUsernames.contains(username.lowercased()) {
                return true
            }
            return false
        }
        return !allBlocked
    default:
        return true
    }
}
```

### Key types/paths for reference
- `Message.author: Peer?` — defined in `submodules/Postbox/Sources/Message.swift`
- `TelegramUser.addressName: String?` — username without @
- `Peer.addressName` — available on `submodules/TelegramCore/Sources/Utils/PeerUtils.swift`
- `ChatHistoryEntry` enum — `submodules/TelegramUI/Components/Chat/ChatHistoryEntry/Sources/ChatHistoryEntry.swift`

### Notification suppression
- Push notifications are handled server-side by Telegram — can't fully suppress without server changes
- For in-app notifications: filter in `NotificationContentContext.swift` (`submodules/TelegramUI/Sources/NotificationContentContext.swift`) by checking message author before displaying

---

## Step 5: Build & Run on iPhone

### Option A: Via Xcode (recommended for development)

1. Open generated `.xcworkspace` in Xcode
2. Select your iPhone as target device
3. Product > Run (Cmd+R)
4. First run: trust the developer certificate on iPhone:
   Settings > General > VPN & Device Management > your dev cert > Trust

### Option B: Via command line (IPA build)

```sh
source ~/.zshrc 2>/dev/null; \
python3 build-system/Make/Make.py --overrideXcodeVersion \
  --cacheDir ~/telegram-bazel-cache \
  build \
  --configurationPath build-system/my-configuration.json \
  --xcodeManagedCodesigning \
  --buildNumber=1 \
  --configuration=debug_arm64
```

Note: use `debug_arm64` for physical device (not `debug_sim_arm64` which is simulator-only).

Then install the IPA via Xcode Devices & Simulators window, or `ios-deploy`.

---

## Step 6 (Optional): Deploy to Hetzner for OTA Distribution

Only needed if distributing to other devices without TestFlight.

- [ ] Build release IPA with ad-hoc provisioning (requires paid Apple Developer account + registered device UDIDs)
- [ ] Provision Hetzner VPS
- [ ] Install Nginx + Certbot (SSL required for OTA)
- [ ] Create `manifest.plist` for OTA install
- [ ] Host `.ipa` + `manifest.plist`
- [ ] Configure domain DNS -> Hetzner IP
- [ ] Install link: `itms-services://?action=download-manifest&url=https://yourdomain.com/manifest.plist`

---

## Execution Order

1. **Create config** — `build-system/my-configuration.json` with your Team ID
2. **Generate Xcode project** — Step 2 command
3. **Verify clean build** — build unmodified app, run on iPhone to confirm setup works
4. **Implement search restriction** — Mod 1 (Step 3)
5. **Build & test** — verify global search hidden, in-chat search works
6. **Implement message filter** — Mod 2 (Step 4)
7. **Build & test** — verify messages from @imaginati8n hidden
8. **Deploy** — Step 6 if needed
