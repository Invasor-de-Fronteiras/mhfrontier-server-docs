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
2. It calls [`loginCog(username, password, password)`](#loginCog) — or `loginHangame`/`loginDmm` for those providers.
3. The game executable communicates with the sign server over the MHF binary protocol.
4. The launcher polls [`getLastAuthResult()`](#getLastAuthResult) and [`getSignResult()`](#getSignResult) until the result is no longer `AUTH_PROGRESS`.
5. On success, [`getCharacterInfo()`](#getCharacterInfo) becomes available and returns the character list XML.
6. The launcher calls [`selectCharacter(charUid, charUid)`](#selectCharacter) and then [`exitLauncher()`](#exitLauncher) to start the game.

To create a new character, the launcher appends `+` to the username and calls `loginCog` again — the server interprets this as a new-character request and returns an uninitialised character slot.

---

## Character Info XML

[`getCharacterInfo()`](#getCharacterInfo) returns an XML document describing all characters for the authenticated account.

```xml
<?xml version='1.0' encoding='shift_jis'?>
<CharacterInfo defaultUid='LAST_PLAYED_UID'>
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
| `uid` | string (hex) | Unique character ID used in [`selectCharacter`](#selectCharacter) |
| `name` | string | Character name in Shift-JIS. For uninitialised characters see [below](#uninitialised-characters). |
| `weapon` | string | Weapon name in Japanese. See [Weapons](#weapons). |
| `HR` | int | Raw HRP value (0–999). See [HR System](#hr-system). |
| `GR` | int | G-rank value. `0` means the character has not entered G-rank. |
| `lastLogin` | int | Unix timestamp of the last time this character was used. |
| `sex` | `M`\|`F` | Character gender. |

Characters are ordered by `lastLogin` descending — the most recently played character is first.

### Uninitialised Characters

A freshly created character that has never been played has `is_new_character = true` in the sign server response. The game executable replaces its name in the XML with a placeholder of `?` characters (typically `?????`). Its weapon attribute also appears as `?????`.

These slots must be filtered out before displaying the character list; they are not selectable — the player must enter the game first to complete character creation.

Detection: name consists entirely of `?` characters **and** weapon is unrecognised.

### HR System

The `HR` attribute carries the raw **HRP** (Hunter Rank Points), an integer 0–999.

| HRP range | Displayed rank |
| --------- | -------------- |
| 0 | HR1 |
| 1 – 29 | HR2 |
| 30 – 49 | HR3 |
| 50 – 98 | HR4 |
| 99 – 298 | HR5 |
| 299 – 997 | HR6 |
| ≥ 998 | HR7 |

HRP = 999 is the sentinel value that marks a character as having reached the G-rank system. When this value is present, `GR` carries the G-rank level and should be displayed instead of HR.

**Entrance hall routing:** After sign-in the game executable routes the player to an entrance hall based on HRP. Players with low HRP land in a beginner hall and cannot see senior-tier channels. If a server wants all players to share a single hall regardless of rank, it must send HRP = 999 for every character — this causes every character to appear as HR7 in the launcher display.

### Weapons

The `weapon` attribute is a Japanese string. Known values:

| Japanese | English |
| -------- | ------- |
| 片手剣 | Sword & Shield |
| 双剣 | Dual Swords |
| 大剣 | Greatsword |
| 太刀 | Longsword |
| ハンマー | Hammer |
| 狩猟笛 | Hunting Horn |
| ランス | Lance |
| ガンランス | Gunlance |
| 穿龍棍 | Tonfa |
| スラッシュアックスＦ | Switch Axe |
| マグネットスパイク | Magnet Spike |
| ヘビィボウガン | Heavy Bowgun |
| ライトボウガン | Light Bowgun |
| 弓 | Bow |

Any unrecognised value (including `?????`) means the weapon is unknown or unset.

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
| [`loginCog`](#loginCog) | `void` | Authenticate against the sign server (JP/TW) |
| [`loginHangame`](#loginHangame) | `void` | Authenticate via Hangame |
| [`loginDmm`](#loginDmm) | `void` | Authenticate via DMM |
| [`getLastAuthResult`](#getLastAuthResult) | `LastAuthResult` | Auth result of the last `loginCog` call |
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
| [`startUpdate`](#startUpdate) | `boolean` | Start the file update process; returns `true` if update is available |
| [`getUpdateStatus`](#getUpdateStatus) | `UpdateStatus` | Current update state |
| [`getUpdatePercentageTotal`](#getUpdatePercentageTotal) | `number` | Overall update progress (0–100) |
| [`getUpdatePercentageFile`](#getUpdatePercentageFile) | `number` | Per-file update progress (0–100) |
| [`deleteCharacter`](#deleteCharacter) | `void` | Delete a character by UID |
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

Plays a built-in launcher audio clip.

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

Enables window dragging while `active` is `true`. Call with `false` to disable.

```js
window.external.beginDrag(true);
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

Opens a URL in the system default browser.

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

Restarts the launcher process (used to switch accounts).

```js
window.external.restartMhf();
```

---

### exitLauncher

Closes the launcher and hands control to the game. Must be called after [`selectCharacter`](#selectCharacter).

```js
window.external.exitLauncher();
```

---

### selectCharacter

Selects the character to play. Both parameters are the same character UID. Must be called before [`exitLauncher`](#exitLauncher).

```js
window.external.selectCharacter(charUid, charUid);
```

---

### loginCog

Initiates authentication against the sign server. The call returns immediately; poll [`getLastAuthResult()`](#getLastAuthResult) to track progress.

```js
window.external.loginCog(username, password, password);
```

To request a new character slot, append `+` to the username:

```js
window.external.loginCog(username + '+', password, password);
```

The sign server creates an uninitialised character row and returns it in the next [`getCharacterInfo()`](#getCharacterInfo) response.

---

### getLastAuthResult

Returns the result of the last `loginCog` call.

```js
const result = window.external.getLastAuthResult();
```

| Value | Meaning |
| ----- | ------- |
| `AUTH_NULL` | No login attempted yet |
| `AUTH_PROGRESS` | Login in progress — keep polling |
| `AUTH_SUCCESS` | Login succeeded |
| `AUTH_ERROR_ACC` | Account not found or wrong password |
| `AUTH_ERROR_NET` | Network error |

---

### getSignResult

Returns the sign-server-specific result of the last `loginCog` call.

```js
const result = window.external.getSignResult();
```

| Value | Meaning |
| ----- | ------- |
| `SIGN_UNKNOWN` | No attempt yet |
| `SIGN_SUCCESS` | Sign server accepted the credentials |
| `SIGN_EPASS` | Password mismatch |

---

### getUserId / getPassword

Returns the username / password used in the last `loginCog` call. Useful for re-authenticating (e.g., calling a secondary API) without asking the user again.

```js
const username = window.external.getUserId();
const password = window.external.getPassword();
```

---

### getServerListXml

Returns the raw server-list XML fetched from the server-information host. See [Character Info XML](#character-info-xml) for structure.

```js
const xml = window.external.getServerListXml();
```

---

### getCharacterInfo

Returns an XML document with all characters for the authenticated account. Only available after a successful login. See [Character Info XML](#character-info-xml) for the full format.

```js
const xml = window.external.getCharacterInfo();
```

---

### getIniLastServerIndex / setIniLastServerIndex

Read and write the index of the server last selected by the user. The value is stored in the game's INI file and persists across sessions.

```js
const idx = window.external.getIniLastServerIndex();
window.external.setIniLastServerIndex(1);
```

---

### getAccountRights

Returns a string representing the account's rights/permissions bitmask.

```js
const rights = window.external.getAccountRights();
```

---

### getMhfBootMode

Returns the current boot mode. Only known value is `'_MHF_NORMAL'`.

```js
const mode = window.external.getMhfBootMode();
```

---

### getMhfMutexNumber

Returns the process mutex number. Used to detect if another instance is already running.

```js
const mutex = window.external.getMhfMutexNumber();
```

---

### getLauncherReturnCode

Returns `'NORMAL'` under ordinary operation.

```js
const code = window.external.getLauncherReturnCode();
```

---

### isEnableSessionId

Returns whether session-ID based authentication is enabled. Return type is unknown.

---

### startUpdate

Checks for and starts a file update. Returns `true` if an update is available and was started, `false` otherwise. Track progress with [`getUpdateStatus`](#getUpdateStatus) and [`getUpdatePercentageTotal`](#getUpdatePercentagetotal--getUpdatePercentagefile).

```js
const hasUpdate = window.external.startUpdate();
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

Return the overall update progress and the current file's progress, both as integers 0–100.

```js
const total = window.external.getUpdatePercentageTotal();
const file  = window.external.getUpdatePercentageFile();
```

---

### deleteCharacter

Deletes a character by UID.

```js
window.external.deleteCharacter(charUid);
```

---

### extractLog

Extracts the game log. Return type is unknown.
