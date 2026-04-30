# Launcher

The MHF launcher is a fixed-size (`1124 × 600 px`) Internet Explorer 11 browser window hosted inside the game executable. It runs JavaScript 1.3 and exposes a set of native functions through `window.external`.

On startup the launcher contacts two servers:

- **Server-information host** (`srv-mhf.capcom-networks.jp`) — fetches the server list.
- **Launcher host** (`cog-members.mhf-z.jp`) — fetches the launcher HTML/JS.

---

## Launcher Hosts

| Name | Host |
| ---- | ---- |
| [Server information](#server-information) | `srv-mhf.capcom-networks.jp` |
| [Launcher HTML](#launcher-html) | `cog-members.mhf-z.jp` |

### Server information

```
Headers:
  host: srv-mhf.capcom-networks.jp
  user-agent: MHF svrsel_trm_pc/1.0
```

#### Routes

| Method | Route | Description |
| ------ | ----- | ----------- |
| GET | `/server/serverlist.php` | Returns the server group list |
| POST | `/server/unique.php` | Checks if a character name is available |

##### GET /server/serverlist.php

Returns an XML list of server groups. Each `<group>` entry is one selectable server.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server_groups>
  <group idx="0" nam="SERVER_NAME" ip="SERVER_IP" port="SERVER_PORT" />
  <group idx="1" nam="SERVER_NAME_2" ip="SERVER_IP_2" port="SERVER_PORT_2" />
</server_groups>
```

This XML is also accessible from JavaScript as [`window.external.getServerListXml()`](#getServerListXml).

### Launcher HTML

```
Headers:
  host: cog-members.mhf-z.jp
  accept: image/gif, image/jpeg, image/pjpeg, ...
  user-agent: Mozilla/5.0 (Windows NT 6.2; WOW64; Trident/7.0; rv:11.0) like Gecko
  accept-language: pt-BR,pt;q=0.8,en-US;q=0.5,en;q=0.2
  accept-encoding: gzip, deflate
```

#### Routes

| Method | Route | Description |
| ------ | ----- | ----------- |
| GET | `/launcher/?ver=2.016` | Returns the launcher HTML |

---

## Authentication Flow

1. The launcher collects username and password from the user.
2. It calls [`loginCog(username, password, anyString)`](#loginCog) — or `loginHangame`/`loginDmm` for those providers.
3. The game executable communicates with the sign server over the MHF binary protocol.
4. The launcher polls [`getLastAuthResult()`](#getLastAuthResult) every ~10 ms until the result is no longer `AUTH_PROGRESS`.
5. On success, [`getCharacterInfo()`](#getCharacterInfo) becomes available and returns the character list XML.
6. The launcher calls [`selectCharacter(charUid, charUid)`](#selectCharacter), waits ~3 seconds, then calls [`exitLauncher()`](#exitLauncher) to start the game.

### Auto-login

The launcher can persist credentials in `localStorage` and automatically call `loginCog` on the next launch. The last selected character UID is also stored, allowing fully automatic login + character selection. Because `localStorage` is backed by the IE cache, clearing the IE cache removes saved credentials and the auto-login state.

### New character

To request a new character slot, the launcher appends `+` to the username and calls `loginCog` again:

```js
window.external.loginCog(username + '+', password, password);
```

The sign server creates an uninitialised character row and returns it in the next [`getCharacterInfo()`](#getCharacterInfo) response.

### Character deletion

Deletion is also asynchronous. The launcher calls [`deleteCharacter(uid)`](#deleteCharacter) then polls `getLastAuthResult()` for `DEL_PROGRESS` → `DEL_SUCCESS`, after which it re-authenticates to refresh the character list.

### Keyboard shortcuts

| Key | Action |
| --- | ------ |
| `Enter` | Submit login form / launch game |
| `,` | Select previous character |
| `.` | Select next character |
| `~` | Open debug eval console |

---

## Character Info XML

[`getCharacterInfo()`](#getCharacterInfo) returns an XML document describing all characters for the authenticated account.

The document uses **single-quoted attributes** and **Shift-JIS encoding**. Before parsing with `DOMParser`, replace `'` with `"` and `&apos;` with `'`.

```xml
<?xml version='1.0' encoding='shift_jis'?>
<CharacterInfo defaultUid='LAST_PLAYED_CHAR_ID'>
  <Character
    uid='CHAR_ID'
    name='CHARACTER_NAME'
    weapon='WEAPON_NAME_JP'
    HR='HRP_VALUE'
    GR='GR_VALUE'
    lastLogin='UNIX_TIMESTAMP'
    sex='M|F'
  />
  ...
</CharacterInfo>
```

### Attribute Reference

| Attribute | Type | Description |
| --------- | ---- | ----------- |
| `defaultUid` (on `CharacterInfo`) | string | UID of the last character the account played. Used to restore the previous selection. |
| `uid` | string (hex) | Unique character ID, passed to [`selectCharacter`](#selectCharacter) and [`deleteCharacter`](#deleteCharacter). |
| `name` | string | Character name in Shift-JIS. For uninitialised characters see [below](#uninitialised-characters). |
| `weapon` | string | Weapon name in Japanese. See [Weapons](#weapons). |
| `HR` | int (0–999) | Raw HRP value. See [HR System](#hr-system). Capped at 999 for display. |
| `GR` | int (0–999) | G-rank value. `0` means the character has not entered G-rank. Capped at 999 for display. |
| `lastLogin` | int | Unix timestamp (seconds) of the last time this character was used. |
| `sex` | `M`\|`F` | Character gender. |

Characters are delivered in the order the sign server returns them. The sign server queries by `last_login DESC`, so the most recently played character is first.

### Uninitialised Characters

A freshly created character that has never been played has `is_new_character = true` set in the sign server response. The game executable replaces the name in the XML with a placeholder consisting entirely of `?` characters (observed: `?????`). The weapon attribute also appears as `?????`.

These slots must be filtered out before displaying the character list — they are not selectable. The player must enter the game once to complete character creation and initialise their save file.

Detection: name consists entirely of `?` characters (any length) **and** weapon is unrecognised.

### HR System

The `HR` attribute is the raw **HRP** (Hunter Rank Points) value stored on the sign server, an integer 0–999. It is **not** a display rank — the launcher is responsible for converting it using the threshold table below if it wants to show a rank label.

| HRP range | Displayed rank |
| --------- | -------------- |
| 0 | HR1 |
| 1 – 29 | HR2 |
| 30 – 49 | HR3 |
| 50 – 98 | HR4 |
| 99 – 298 | HR5 |
| 299 – 997 | HR6 |
| ≥ 998 | HR7 |

HRP = 999 is the sentinel value that marks a character as having transitioned to the G-rank system. When this value is present, `GR` carries the G-rank level and should be shown instead of HR.

**Entrance hall routing:** After sign-in the game executable routes the player to an entrance hall based on HRP. Players with low HRP land in a beginner hall and cannot see senior-tier channels. If a server wants all players to share a single hall regardless of rank, it sends HRP = 999 for every character. This does not affect the `GR` value.

### Weapons

The `weapon` attribute is a Japanese string. Known values:

| Japanese | English | Icon filename |
| -------- | ------- | ------------- |
| 片手剣 | Sword & Shield | `ss` |
| 双剣 | Dual Swords | `db` |
| 大剣 | Greatsword | `gs` |
| 太刀 | Longsword | `ls` |
| ハンマー | Hammer | `hm` |
| 狩猟笛 | Hunting Horn | `hh` |
| ランス | Lance | `ln` |
| ガンランス | Gunlance | `gl` |
| 穿龍棍 | Tonfa | `tf` |
| スラッシュアックスＦ | Switch Axe F | `sa` |
| マグネットスパイク | Magnet Spike | `ms` |
| ヘビィボウガン | Heavy Bowgun | `hbg` |
| ライトボウガン | Light Bowgun | `lbg` |
| 弓 | Bow | `bow` |

Any unrecognised value (including `?????`) means the weapon is unknown or unset. Icon: `uk`.

---

## `window.external` API

All native functions are exposed on `window.external`. They throw on failure and can be caught:

```js
try {
  window.external.getLastAuthResult();
} catch (err) {
  // handle error
}
```

### Full Function List

| Function | Returns | Description |
| -------- | ------- | ----------- |
| [`playSound`](#playSound) | `void` | Play a built-in launcher sound |
| [`beginDrag`](#beginDrag) | `void` | Allow the user to drag the window |
| [`minimizeWindow`](#minimizeWindow) | `void` | Minimize the window |
| [`closeWindow`](#closeWindow) | `void` | Close the launcher window |
| [`openBrowser`](#openBrowser) | `void` | Open a URL in the system browser |
| [`openMhlConfig`](#openMhlConfig) | `void` | Open the MHL configuration dialog |
| [`restartMhf`](#restartMhf) | `void` | Restart the launcher |
| [`exitLauncher`](#exitLauncher) | `void` | Close the launcher and start the game |
| [`selectCharacter`](#selectCharacter) | `void` | Select which character to play |
| [`deleteCharacter`](#deleteCharacter) | `void` | Delete a character by UID (asynchronous) |
| [`loginCog`](#loginCog) | `void` | Authenticate against the sign server (JP/TW) |
| [`loginHangame`](#loginHangame) | `void` | Authenticate via Hangame |
| [`loginDmm`](#loginDmm) | `void` | Authenticate via DMM |
| [`getLastAuthResult`](#getLastAuthResult) | `LastAuthResult` | Result of the last auth or delete operation |
| [`getSignResult`](#getSignResult) | `SignResult` | Sign-server result of the last `loginCog` call |
| [`getUserId`](#getUserId) | `string` | Username used for the last login |
| [`getPassword`](#getPassword) | `string` | Password used for the last login |
| [`getServerListXml`](#getServerListXml) | `string` | Raw XML from the server-info host |
| [`getCharacterInfo`](#getCharacterInfo) | `string` | XML with character data (available after auth success) |
| [`getAccountRights`](#getAccountRights) | `string` | Account rights/permissions bitmask |
| [`getMhfBootMode`](#getMhfBootMode) | `BoostModeTypes` | Current boot mode |
| [`getMhfMutexNumber`](#getMhfMutexNumber) | `number` | Process mutex number |
| [`getIniLastServerIndex`](#getIniLastServerIndex) | `number` | Index of the server last selected by the user |
| [`setIniLastServerIndex`](#setIniLastServerIndex) | `void` | Persist the user's server selection |
| [`getLauncherReturnCode`](#getLauncherReturnCode) | `'NORMAL'` | Return code from the launcher process |
| [`isEnableSessionId`](#isEnableSessionId) | `unknown` | Whether session ID auth is enabled |
| [`startUpdate`](#startUpdate) | `boolean` | Start the file update process |
| [`getUpdateStatus`](#getUpdateStatus) | `UpdateStatus` | Current update state |
| [`getUpdatePercentageTotal`](#getUpdatePercentageTotal) | `number` | Overall update progress (0–100) |
| [`getUpdatePercentageFile`](#getUpdatePercentageFile) | `number` | Per-file update progress (0–100) |
| [`extractLog`](#extractLog) | `unknown` | Extract the game log |
| `debugGetIniUserId` | `string` | Debug: read userId from INI |
| `debugGetIniPassword` | `string` | Debug: read password from INI |

---

### TypeScript Declarations

```ts
declare global {
  interface External {
    playSound(song: LauncherSongs): void;
    beginDrag(active: boolean): void;
    openBrowser(url: string): void;
    restartMhf(): void;
    minimizeWindow(): void;
    openMhlConfig(): void;
    closeWindow(): void;
    getAccountRights(): string;
    getMhfBootMode(): BoostModeTypes;
    getIniLastServerIndex(): number;
    setIniLastServerIndex(idx: number): void;
    getServerListXml(): string;
    getMhfMutexNumber(): number;
    getUserId(): string;
    getPassword(): string;
    getLastAuthResult(): LastAuthResult;
    getSignResult(): SignResult;
    isEnableSessionId(): unknown;
    getCharacterInfo(): string;
    extractLog(): unknown;
    getLauncherReturnCode(): 'NORMAL';
    selectCharacter(charUid: string, charUid1: string): void;
    exitLauncher(): void;
    loginCog(username: string, password: string, confirmPassword: string): void;
    startUpdate(): boolean;
    getUpdatePercentageTotal(): number;
    getUpdatePercentageFile(): number;
    getUpdateStatus(): UpdateStatus;
    deleteCharacter(charUid: string): void;
  }
}

export type LauncherSongs = 'IDR_WAV_SEL' | 'IDR_WAV_OK' | 'IDR_WAV_PRE_LOGIN' | 'IDR_NIKU';
export type BoostModeTypes = '_MHF_NORMAL';

export enum LastAuthResult {
  None = 'AUTH_NULL',
  AuthSuccess = 'AUTH_SUCCESS',
  InLoading = 'AUTH_PROGRESS',
  AuthErrorAcc = 'AUTH_ERROR_ACC',
  AuthErrorNet = 'AUTH_ERROR_NET',
  DeleteInProgress = 'DEL_PROGRESS',
  DeleteSuccess = 'DEL_SUCCESS',
}

export enum SignResult {
  None = 'SIGN_UNKNOWN',
  SignSuccess = 'SIGN_SUCCESS',
  NotMatchPassword = 'SIGN_EPASS',
}

export enum UpdateStatus {
  None = '0',
  UpdateStart = 'UM_UPDATE_START',
  UpdateOk = 'UM_UPDATE_OK',
}
```

---

### playSound

Plays a built-in launcher audio clip. Some sounds can also be loaded from local audio files when the built-in is unavailable.

```js
window.external.playSound('IDR_NIKU');
```

| ID | Plays when |
| -- | ---------- |
| `IDR_WAV_SEL` | Mouse hovers over an element |
| `IDR_WAV_OK` | An action succeeds |
| `IDR_WAV_PRE_LOGIN` | User clicks the login button |
| `IDR_NIKU` | Login is successful |

---

### beginDrag

Enables window dragging while `active` is `true`. Called on `mousedown` and `mouseup` events on draggable regions, and with `false` on `mouseover` events over interactive elements to prevent accidental dragging.

```js
window.external.beginDrag(true);   // start drag
window.external.beginDrag(false);  // stop drag
```

---

### minimizeWindow

Minimizes the launcher window.

```js
window.external.minimizeWindow();
```

---

### closeWindow

Closes the launcher window without starting the game.

```js
window.external.closeWindow();
```

---

### openBrowser

Opens a URL in the system default browser. Typically guarded by a confirmation modal before calling.

```js
window.external.openBrowser('https://example.com');
```

---

### openMhlConfig

Opens the native MHL configuration dialog.

```js
window.external.openMhlConfig();
```

---

### restartMhf

Restarts the launcher process. Used to log out and switch accounts.

```js
window.external.restartMhf();
```

---

### exitLauncher

Closes the launcher and hands control to the game. Must be called after [`selectCharacter`](#selectCharacter). A ~3 second delay between `selectCharacter` and `exitLauncher` is recommended to allow the game to process the selection.

```js
window.external.selectCharacter(uid, uid);
setTimeout(function () {
  window.external.exitLauncher();
}, 3000);
```

---

### selectCharacter

Marks a character as the one to play. Both arguments receive the same character UID. Must be called before [`exitLauncher`](#exitLauncher).

```js
window.external.selectCharacter(charUid, charUid);
```

---

### deleteCharacter

Initiates asynchronous deletion of a character. After calling this, poll [`getLastAuthResult()`](#getLastAuthResult) for `DEL_PROGRESS` → `DEL_SUCCESS`, then re-authenticate to refresh the character list.

```js
window.external.deleteCharacter(charUid);

function checkDelete() {
  var result = window.external.getLastAuthResult();
  if (result == 'DEL_PROGRESS') {
    setTimeout(checkDelete, 10);
  } else if (result == 'DEL_SUCCESS') {
    // re-authenticate and refresh
  }
}
checkDelete();
```

---

### loginCog

Initiates authentication against the sign server. Returns immediately; poll [`getLastAuthResult()`](#getLastAuthResult) every ~10 ms to track progress. The third argument (confirm password) is not validated by the server and can be any string.

```js
window.external.loginCog(username, password, password);
```

To request a new character slot, append `+` to the username:

```js
window.external.loginCog(username + '+', password, password);
```

---

### getLastAuthResult

Returns the result of the last `loginCog` or [`deleteCharacter`](#deleteCharacter) call. Shared between auth and deletion flows.

```js
var result = window.external.getLastAuthResult();
```

| Value | Meaning |
| ----- | ------- |
| `AUTH_NULL` | No login attempted yet |
| `AUTH_PROGRESS` | Login in progress — keep polling |
| `AUTH_SUCCESS` | Login succeeded |
| `AUTH_ERROR_ACC` | Account not found or wrong password |
| `AUTH_ERROR_NET` | Network error |
| `DEL_PROGRESS` | Character deletion in progress |
| `DEL_SUCCESS` | Character deletion succeeded |

---

### getSignResult

Returns the sign-server-specific result of the last `loginCog` call.

```js
var result = window.external.getSignResult();
```

| Value | Meaning |
| ----- | ------- |
| `SIGN_UNKNOWN` | No attempt yet |
| `SIGN_SUCCESS` | Sign server accepted the credentials |
| `SIGN_EPASS` | Password mismatch |

---

### getUserId / getPassword

Returns the username / password used in the last `loginCog` call. Useful for re-authenticating without prompting the user again.

```js
var username = window.external.getUserId();
var password = window.external.getPassword();
```

---

### getServerListXml

Returns the raw server-list XML fetched from the server-information host. See [Server information](#server-information) for the XML structure.

```js
var xml = window.external.getServerListXml();
```

---

### getCharacterInfo

Returns the character list XML for the authenticated account. Only available after a successful login. See [Character Info XML](#character-info-xml) for the full format, including the single-quote preprocessing required before parsing.

```js
var xml = window.external.getCharacterInfo();
// Normalise quotes before parsing
xml = xml.split("'").join('"').split('&apos;').join("'");
var doc = new DOMParser().parseFromString(xml, 'text/xml');
```

---

### getIniLastServerIndex / setIniLastServerIndex

Read and write the index of the server last selected by the user. The value is stored in the game's INI file and persists across sessions.

```js
var idx = window.external.getIniLastServerIndex();
window.external.setIniLastServerIndex(1);
```

---

### getAccountRights

Returns a string representing the account's rights/permissions bitmask.

```js
var rights = window.external.getAccountRights();
```

---

### getMhfBootMode

Returns the current boot mode. Only known value is `'_MHF_NORMAL'`.

```js
var mode = window.external.getMhfBootMode();
```

---

### getMhfMutexNumber

Returns the process mutex number. Used to detect if another game instance is already running.

```js
var mutex = window.external.getMhfMutexNumber();
```

---

### getLauncherReturnCode

Returns `'NORMAL'` under ordinary operation.

```js
var code = window.external.getLauncherReturnCode();
```

---

### isEnableSessionId

Returns whether session-ID based authentication is enabled. Return type is unknown.

---

### startUpdate

Checks for available updates and starts the download if one is found. Returns `true` if an update was started. Track progress with [`getUpdateStatus`](#getUpdateStatus) and [`getUpdatePercentageTotal / getUpdatePercentageFile`](#getUpdatePercentageTotal--getUpdatePercentageFile).

```js
var hasUpdate = window.external.startUpdate();
```

---

### getUpdateStatus

Returns the current update state.

| Value | Meaning |
| ----- | ------- |
| `'0'` | No update in progress |
| `'UM_UPDATE_START'` | Update downloading |
| `'UM_UPDATE_OK'` | Update complete |

---

### getUpdatePercentageTotal / getUpdatePercentageFile

Return the overall update progress and the current-file progress, both as integers 0–100.

```js
var total = window.external.getUpdatePercentageTotal();
var file  = window.external.getUpdatePercentageFile();
```

---

### extractLog

Extracts the game log. Return type is unknown.
