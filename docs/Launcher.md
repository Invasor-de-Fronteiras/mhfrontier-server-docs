# Launcher

The MHF launcher is a fixed-size Internet Explorer 11 browser window hosted inside the game executable. It runs JavaScript 1.3 and exposes a set of native functions through `window.external`.

PC window size: **1124 × 600 px**.  
PS Vita window size: **840 × 440 px**.

On startup the launcher contacts two servers:

- **Server-information host** (`srv-mhf.capcom-networks.jp`) — fetches the server list.
- **Launcher host** (`cog-members.mhf-z.jp`) — fetches the launcher HTML/JS.

These hostnames are hardcoded in `mhl.dll`. To intercept them, the client machine must redirect those exact domains to your server. Open `C:\Windows\System32\drivers\etc\hosts` as administrator and add:

```
127.0.0.1 srv-mhf.capcom-networks.jp
127.0.0.1 cog-members.mhf-g.jp cog-members.mhf-z.jp
127.0.0.1 erupe.custom
127.0.0.1 launcher.arcamh.com
```

The domains must match exactly what is declared in your `mhl.dll` — different builds may use different hostnames.

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
| GET | `/server/serverlist.xml` | Returns the server group list (TW client) |
| GET | `/serverlist.xml` | Returns the server group list (JP client) |
| POST | `/server/unique.php` | Checks if a character name is available |

##### GET /server/serverlist.xml

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
| GET | `/bnr/launcher.html` | Banner rotation HTML (see [Launcher HTML File Formats](#launcher-html-file-formats)) |
| GET | `/launcher_list.html` | News list HTML (see [Launcher HTML File Formats](#launcher-html-file-formats)) |
| GET | `/version` | Returns the current patch version as a plain string |
| POST | `/auth/launcher/login` | COG short-lived auth endpoint (see [JP Auth Endpoint](#jp-auth-endpoint)) |

##### GET /version

Returns the current client version as a plain string (e.g. `2.016`). The launcher calls this before showing the update screen. If the endpoint returns 404 or is unreachable, the update step is skipped.

##### POST /auth/launcher/login

Called by a hidden iframe inside the launcher as part of the COG authentication flow (see [PC (COG)](#pc-cog)). The iframe posts the user's credentials and expects an HTML page that calls `parent.postMessage(result, origin)` with a JSON payload.

The response must be an HTML page that executes `postMessage` on load:

```html
<!DOCTYPE html>
<html>
<body onload="doPost();">
<script>
function doPost() {
  parent.postMessage(document.getElementById("result").getAttribute("value"), "http://cog-members.mhf-z.jp");
}
</script>
<input id="result" value='{RESULT_JSON}'/>
</body>
</html>
```

Where `{RESULT_JSON}` is:

```json
{"result": "Ok", "skey": "<password>", "code": "000", "msg": ""}
```

The `skey` field must carry the raw password. The game executable reads this value and passes it directly to the sign server as the authentication credential. Returning `code: "000"` tells the launcher to proceed to `loginCog`. Any other code triggers an error dialog (see [COG short-lived auth service response codes](#cog-short-lived-auth-service-response-codes)).

---

## Launcher HTML File Formats

Both `bnr/launcher.html` and `launcher_list.html` use the same delimiter convention. The launcher fetches them with `$.ajax({ cache: false })` and splits the response body on the string `<!--###content###-->`. The result must have exactly **3 parts** — anything else causes the launcher to fall back to opening the URL in the system browser.

```
[preamble]<!--###content###-->[content items]<!--###content###-->[postamble]
```

Only the middle section (index 1) is used. Everything outside the delimiters is ignored.

### bnr/launcher.html — Banner rotation

The middle section must be a `<ul>` containing `<li>` items, one per banner slide. Maximum 5 banners are displayed.

```html
<!--###content###-->
<li><a href="/sp/news/123.html"><img src="images/bnr/example.jpg" alt="..." /></a></li>
<li><a href="http://external.example.com/page"><img src="images/bnr/example2.jpg" alt="..." /></a></li>
<!--###content###-->
```

The launcher rewrites `src=` paths to absolute URLs based on the page's host. Internal paths (starting with `/`) are kept as-is; relative paths are prefixed with the banner CDN base. Banners rotate automatically every **5000 ms**. If an image fails to load, the launcher retries with a cache-busting query string (`?c={n}`) every 5000 ms.

Items with CSS class filter suffixes (`cogHide`, `cogOnly`, `nhnOnly`, `dmmOnly`, etc.) are hidden or shown depending on the authentication mode.

### launcher_list.html — News list

The middle section is raw HTML inserted directly into the news list pane. Links pointing to `/sp/news/` paths are intercepted by the launcher and loaded inline (see below); all other links open in the system browser.

### /sp/news/{id} — News article detail

Individual article pages follow the same three-part delimiter format. The middle section must contain the article body HTML. The launcher strips `<script>`, `<iframe>`, `<video>`, `<audio>`, `<source>`, and `<input>` tags from the content before rendering.

---

## Authentication Providers

The launcher determines which authentication provider to use from the page's hostname:

| Hostname contains | Provider | Login call |
| ----------------- | -------- | ---------- |
| `capcom-onlinegames.jp` | COG (Capcom Online Games) | `loginCog` |
| `hangame-` | Hangame (NHN) | `loginHangame` |
| anything else | DMM | `loginDmm` |

The mode is detected once at page load. COG is the default for the official Japanese service. Hangame and DMM use native browser windows for their OAuth flow — the JavaScript has no access to the credentials.

---

## Authentication Flow

### PC (COG)

1. The launcher collects username and password from the user.
2. A hidden iframe is created pointing to `www.capcom-onlinegames.jp/auth/bnr/launcher.html?q={counter}`. That page sends `{"action":"standby"}` to the parent via `postMessage` when ready. The launcher responds with the credentials payload:
   ```json
   {"id": "<username>", "pw": "<password>", "svid": "<server_id>", "lifetime": "60", "action": "login"}
   ```
   The auth page then POSTs `pw` to `POST /auth/launcher/login` and relays the server's response back to the launcher via `postMessage`.
3. The auth service responds with a result code. `000` = proceed; anything else = show error.
4. On `000`, the launcher calls [`loginCog(username, password, password)`](#loginCog).
5. The game executable communicates with the sign server over the MHF binary protocol.
6. The launcher polls [`getLastAuthResult()`](#getLastAuthResult) every **1000 ms** until the result is no longer `AUTH_PROGRESS`. A **60-second backbone timeout** fires if no response is received, showing an error dialog.
7. On success, the launcher calls [`getAccountRights()`](#getAccountRights). If the account has no `trial` right, a registration dialog is shown. If HR ≥ 100 and no `basic` right (Hunter Life Course), a purchase dialog is shown.
8. [`getCharacterInfo()`](#getCharacterInfo) becomes available and returns the character list XML.
9. The launcher calls [`selectCharacter(charUid, charUid)`](#selectCharacter), then calls [`exitLauncher()`](#exitLauncher) after **500 ms**.

### PC (Hangame / DMM)

Same polling loop as COG, but steps 2–4 are replaced by a call to [`loginHangame()`](#loginHangame) or [`loginDmm()`](#loginDmm). The native window handles credentials; JavaScript never sees them.

### PS3 / PS Vita

Console authentication uses a dedicated COG account linking page (`link.html`) with console-specific bridge functions. See [Console Launchers (PS3 / PS Vita)](#console-launchers-ps3--ps-vita).

### COG short-lived auth service response codes

Codes returned by the hidden iframe to the launcher before `loginCog` is called:

| Code | Meaning |
| ---- | ------- |
| `000` | Success — proceed to `loginCog` |
| `102` | Server busy — retry later |
| `200` | Character encoding not supported |
| `201`, `300`, `302`, `320` | Wrong ID / password |
| `301` | Account temporarily suspended (1 hour) |
| `304` | Too many wrong password attempts |
| `321` | Security card data mismatch |
| `202`, `220`, `221`, `227`, `777` | Permission denied / invalid access |
| `235`, `309`, `360` | Unknown error |
| `9000` | COG server maintenance |

### Auto-login

The launcher can persist credentials in browser cookies (`cogid{MUTEX}` for username, `pw{MUTEX}` for password, keyed by the process mutex number) and automatically call `loginCog` on the next launch. The last selected character UID is also stored, allowing fully automatic login + character selection. Clearing the IE cache removes saved credentials and the auto-login state.

### New character

To request a new character slot, the launcher appends `+` to the username and calls `loginCog` again:

```js
window.external.loginCog(username + '+', password, password);
```

The sign server creates an uninitialised character row and returns it in the next [`getCharacterInfo()`](#getCharacterInfo) response.

### Character deletion

Deletion is asynchronous. The launcher calls [`deleteCharacter(uid)`](#deleteCharacter) then polls `getLastAuthResult()` for `DEL_PROGRESS` → `DEL_SUCCESS`, after which it re-authenticates to refresh the character list. The official client uses a three-step confirmation dialog that requires the player to type the character's UID before deletion is allowed.

Attempting to delete the last character on an account triggers a 7-day cooldown before a replacement guaranteed character is provided.

### Keyboard shortcuts

| Key | Action |
| --- | ------ |
| `Enter` | Submit login form / launch game |
| `,` | Select previous character |
| `.` | Select next character |
| `~` | Open debug eval console (debug builds only) |

---

## Server List XML

[`getServerListXml()`](#getServerListXml) returns the full server list XML. The game uses single-quoted attributes when sending this to console clients.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server_groups>
  <group idx='0' nam='Select a Server' ip='' />
  <group idx='1' nam='Localhost' cog='pre' ip='127.0.0.1' port='53312' svid='1000' />
  <group idx='2' nam='Production' cog='c'   ip='203.0.113.4' port='53312' svid='2000' />
</server_groups>
```

### Server Group Attributes

| Attribute | Type | Description |
| --------- | ---- | ----------- |
| `idx` | int | Zero-based selection index. Index 0 is typically a placeholder entry. |
| `nam` | string | Display name shown in the server dropdown. |
| `ip` | string | Server IP or hostname. **Empty string `""`** marks the server as blocked — it cannot be selected. |
| `port` | int | Sign server port (default `53312`). |
| `cog` | string | COG environment. `pre` = pre-release/staging, `c` = production. |
| `svid` | string | Server ID used for backend routing. Defaults to `"1000"` if absent. Server ID `"1018"` triggers a COOP restriction alert. Server names containing `④` or `XBOX` also trigger the COOP alert regardless of `svid`. |
| `oauth_url` | string | Console-only. URL for the COG account linking page (`link.html`). |

The launcher stores the last selected index via [`setIniLastServerIndex()`](#getIniLastServerIndex--setIniLastServerIndex).

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
| `HR` | int (0–999) | Raw HRP value. See [HR System](#hr-system). |
| `GR` | int (0–999) | G-rank value. `0` means the character has not entered G-rank. |
| `lastLogin` | int | Unix timestamp (seconds) of the last time this character was used. |
| `sex` | `M`\|`F` | Character gender. |

Characters are delivered in the order the sign server returns them. The sign server queries by `last_login DESC`, so the most recently played character is **first**.

The launcher enforces a maximum of **11 characters** per account. The "Add character" button is disabled when the count reaches 11.

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

**Display format:** `HR{hrp}` for normal characters. If GR > 0, the launcher appends `　GR{gr}` (full-width space separator). Example: `HR42　GR7`.

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

## Account Rights

[`getAccountRights()`](#getAccountRights) returns XML listing the rights attached to the account:

```xml
<rights>
  <right name="trial" />
  <right name="basic" />
</rights>
```

| Right name | Meaning |
| ---------- | ------- |
| `trial` | Account can access the game servers |
| `basic` | Account has the Hunter Life Course subscription (required to play characters with HRP ≥ 100) |

If neither right is present the account must register. If `trial` is present but not `basic`, the player can play but will be prompted to purchase the subscription when attempting to launch with a high-HRP character.

---

## Console Launchers (PS3 / PS Vita)

PS3 and PS Vita use a stripped-down launcher (`link.html`) dedicated to linking a COG account. The main character-select UI is embedded in the game executable itself on consoles. Protocol-level, PS3 and Vita are identical.

### Console bridge functions

In addition to the standard `window.external` API, consoles expose:

| Function | Description |
| -------- | ----------- |
| `mhfBrowserGetCOGId()` | Returns the cached COG username |
| `mhfBrowserSetCOGId(id)` | Stores a COG username to the console's local storage |
| `mhfBrowserGetCOGPass()` | Returns the cached COG password |
| `mhfBrowserSetCOGPass(pw)` | Stores a COG password to the console's local storage |
| `endCogAuth(payload)` | Finalises authentication. `payload` = `username + "\n" + password` |

The `link.html` page reads cached credentials on load (pre-filling the form), then calls these on submit:

```js
window.external.mhfBrowserSetCOGId(user.value);
window.external.mhfBrowserSetCOGPass(pass.value);
window.external.endCogAuth(user.value + "\n" + pass.value);
```

### PS3 sign protocol

PS3 clients send a different packet type than PC:

| | PC | PS3 |
| - | -- | --- |
| Packet type | `DSGN:100` | `PS3SGN:100` |
| Auth credential | username + bcrypt password | PSN ID (looked up in `users.psn_id`) |
| Password check | bcrypt | skipped entirely |
| Patch server URLs | conditional | always provided (flag = 2) |
| Patch/entrance hostname | IP or custom domain | `ps3-{language}.zerulight.cc` |
| PSN ID in response | not included | 20-byte Shift-JIS padded string appended after filters |

#### PS3 request layout

```
null_terminated_bytes  "0000000255"   // PS3 identifier
bytes[2]               0x21 0x00      // "!" marker
bytes[82]              padding
null_terminated_bytes  {PSN_ID}       // PSN account ID
```

The server looks up the PSN ID in `users.psn_id` to find the corresponding account. If no match exists the login fails.

### Vita sign protocol

Identical to PS3 except the packet type is `VITASGN:100` and the UA string is `MHF-VITA` instead of `MHF-PS3`.

### PSN ID registration

Players can link their PSN ID in-game via chat command:

```
/psn <psn_id>
```

The server stores the value in `users.psn_id`. From that point the console client can authenticate using only the PSN ID — no password is checked.

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
| [`loginCog`](#loginCog) | `void` | Authenticate against the sign server (COG/JP/TW) |
| [`loginHangame`](#loginHangame) | `void` | Authenticate via Hangame |
| [`loginDmm`](#loginDmm) | `void` | Authenticate via DMM |
| [`getLastAuthResult`](#getLastAuthResult) | `LastAuthResult` | Result of the last auth or delete operation |
| [`getSignResult`](#getSignResult) | `SignResult` | Sign-server result of the last `loginCog` call |
| [`getUserId`](#getUserId--getPassword) | `string` | Username used for the last login |
| [`getPassword`](#getUserId--getPassword) | `string` | Password used for the last login |
| [`getServerListXml`](#getServerListXml) | `string` | Raw XML from the server-info host |
| [`getCharacterInfo`](#getCharacterInfo) | `string` | XML with character data (available after auth success) |
| [`getAccountRights`](#getAccountRights) | `string` | Account rights XML |
| [`getMhfBootMode`](#getMhfBootMode) | `BootModeType` | Current boot mode |
| [`getMhfMutexNumber`](#getMhfMutexNumber) | `number` | Process mutex number |
| [`getIniLastServerIndex`](#getIniLastServerIndex--setIniLastServerIndex) | `number` | Index of the server last selected by the user |
| [`setIniLastServerIndex`](#getIniLastServerIndex--setIniLastServerIndex) | `void` | Persist the user's server selection |
| [`getLauncherReturnCode`](#getLauncherReturnCode) | `string` | Return code from the launcher process |
| [`isEnableSessionId`](#isEnableSessionId) | `boolean` | Whether the session is valid (maintenance check) |
| [`startUpdate`](#startUpdate) | `boolean` | Start the file update process |
| [`getUpdateStatus`](#getUpdateStatus) | `UpdateStatus` | Current update state |
| [`getUpdatePercentageTotal`](#getUpdatePercentageTotal--getUpdatePercentageFile) | `number` | Overall update progress (0–100) |
| [`getUpdatePercentageFile`](#getUpdatePercentageTotal--getUpdatePercentageFile) | `number` | Per-file update progress (0–100) |
| [`extractLog`](#extractLog) | `string` | Extract the game log |
| `debugGetIniUserId` | `string` | Debug: read userId from INI (branch builds only) |
| `debugGetIniPassword` | `string` | Debug: read password from INI (branch builds only) |

---

### TypeScript Declarations

```ts
declare global {
  interface External {
    playSound(song: LauncherSounds): void;
    beginDrag(active: boolean): void;
    openBrowser(url: string): void;
    restartMhf(): void;
    minimizeWindow(): void;
    openMhlConfig(): void;
    closeWindow(): void;
    getAccountRights(): string;
    getMhfBootMode(): BootModeType;
    getIniLastServerIndex(): number;
    setIniLastServerIndex(idx: number): void;
    getServerListXml(): string;
    getMhfMutexNumber(): number;
    getUserId(): string;
    getPassword(): string;
    getLastAuthResult(): LastAuthResult;
    getSignResult(): SignResult;
    isEnableSessionId(): boolean;
    getCharacterInfo(): string;
    extractLog(): string;
    getLauncherReturnCode(): 'NULL' | 'NORMAL' | 'SELFUP' | 'ERR';
    selectCharacter(charUid: string, charUid1: string): void;
    exitLauncher(): void;
    loginCog(username: string, password: string, confirmPassword: string): void;
    loginHangame(): void;
    loginDmm(): void;
    deleteCharacter(charUid: string): void;
    startUpdate(): boolean;
    getUpdatePercentageTotal(): number;
    getUpdatePercentageFile(): number;
    getUpdateStatus(): UpdateStatus;
  }
}

export type LauncherSounds =
  | 'IDR_WAV_SEL'
  | 'IDR_WAV_OK'
  | 'IDR_WAV_PRE_LOGIN'
  | 'IDR_WAV_LOGIN'
  | 'IDR_NIKU'
  | 'IDR_SILENCE';

export type BootModeType =
  | '_MHF_NORMAL'
  | '_MHF_SELFUP'
  | '_MHF_DMM_SELF_UPDATE'
  | '_MHF_AUTOLC'
  | '_MHF_DMM_AUTO_LAUNCH';

export enum LastAuthResult {
  None              = 'AUTH_NULL',
  InProgress        = 'AUTH_PROGRESS',
  AuthSuccess       = 'AUTH_SUCCESS',
  AuthErrorAcc      = 'AUTH_ERROR_ACC',
  AuthErrorPwd      = 'AUTH_ERROR_PWD',
  AuthErrorNet      = 'AUTH_ERROR_NET',
  DeleteInProgress  = 'DEL_PROGRESS',
  DeleteSuccess     = 'DEL_SUCCESS',
  DeleteErrorNet    = 'DEL_ERROR_NET',
  DeleteErrorIvl    = 'DEL_ERROR_IVL',
  DeleteErrorMnc    = 'DEL_ERROR_MNC',
}

export enum SignResult {
  None            = 'SIGN_UNKNOWN',
  Success         = 'SIGN_SUCCESS',
  Failed          = 'SIGN_EFAILED',
  Illegal         = 'SIGN_EILLEGAL',
  Alert           = 'SIGN_EALERT',
  AlertCoop       = 'SIGN_EALERT_COOP',
  Abort           = 'SIGN_EABORT',
  Response        = 'SIGN_ERESPONSE',
  Database        = 'SIGN_EDATABASE',
  WrongPassword   = 'SIGN_EPASS',
  Suspended       = 'SIGN_ESUSPEND',
  Eliminated      = 'SIGN_EELIMINATE',
  Closed          = 'SIGN_ECLOSE',
  ClosedEx        = 'SIGN_ECLOSE_EX',
  NotReady        = 'SIGN_ENOTREADY',
  AlreadyLoggedIn = 'SIGN_EALREADY',
  IpBlocked       = 'SIGN_EIPADDR',
  NoRights        = 'SIGN_ERIGHT',
  AppError        = 'SIGN_EAPP',
  Other           = 'SIGN_EOTHER',
  // Console-specific
  CogLink         = 'SIGN_ECOGLINK',
  CogCode         = 'SIGN_ECOGCODE',
  Token           = 'SIGN_ETOKEN',
  Maintenance     = 'SIGN_EMAINTE',
}

export enum UpdateStatus {
  None        = '0',
  UpdateStart = 'UM_UPDATE_START',
  UpdateOk    = 'UM_UPDATE_OK',
  UpdateNg    = 'UM_UPDATE_NG',
}
```

---

### playSound

Plays a built-in launcher audio clip.

```js
window.external.playSound('IDR_WAV_LOGIN');
```

| ID | When it plays |
| -- | ------------- |
| `IDR_WAV_SEL` | Mouse hovers over an element |
| `IDR_WAV_OK` | An action succeeds / button click |
| `IDR_WAV_PRE_LOGIN` | User clicks the login button |
| `IDR_WAV_LOGIN` | Login succeeds |
| `IDR_NIKU` | Update complete |
| `IDR_SILENCE` | Mute / no sound |

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

Closes the launcher and hands control to the game. Must be called after [`selectCharacter`](#selectCharacter). The official client waits **500 ms** between `selectCharacter` and `exitLauncher`.

```js
window.external.selectCharacter(uid, uid);
setTimeout(function () {
  window.external.exitLauncher();
}, 500);
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

Initiates authentication against the sign server. Returns immediately; poll [`getLastAuthResult()`](#getLastAuthResult) every ~10 ms to track progress. The third argument mirrors the password and is not validated by the server.

```js
window.external.loginCog(username, password, password);
```

To request a new character slot, append `+` to the username:

```js
window.external.loginCog(username + '+', password, password);
```

---

### loginHangame

Initiates Hangame authentication. Opens a native Hangame login window; JavaScript has no access to the credentials. Poll [`getLastAuthResult()`](#getLastAuthResult) as with `loginCog`.

```js
window.external.loginHangame();
```

---

### loginDmm

Initiates DMM authentication. Opens a native DMM login window; JavaScript has no access to the credentials. Poll [`getLastAuthResult()`](#getLastAuthResult) as with `loginCog`.

```js
window.external.loginDmm();
```

---

### getLastAuthResult

Returns the result of the last `loginCog`, `loginHangame`, `loginDmm`, or [`deleteCharacter`](#deleteCharacter) call. Shared between auth and deletion flows.

```js
var result = window.external.getLastAuthResult();
```

| Value | Meaning |
| ----- | ------- |
| `AUTH_NULL` | No login attempted yet |
| `AUTH_PROGRESS` | Login in progress — keep polling |
| `AUTH_SUCCESS` | Login succeeded |
| `AUTH_ERROR_ACC` | Account not found |
| `AUTH_ERROR_PWD` | Password mismatch |
| `AUTH_ERROR_NET` | Network error |
| `DEL_PROGRESS` | Character deletion in progress |
| `DEL_SUCCESS` | Character deletion succeeded |
| `DEL_ERROR_NET` | Network error during deletion |
| `DEL_ERROR_IVL` | Character invalid or already deleted |
| `DEL_ERROR_MNC` | Deletion blocked — 7-day cooldown or maintenance |

---

### getSignResult

Returns the sign-server-specific result of the last `loginCog` call. Used to show a detailed error message when `getLastAuthResult()` returns `AUTH_ERROR_ACC` or `AUTH_ERROR_PWD`.

```js
var result = window.external.getSignResult();
```

| Value | Meaning |
| ----- | ------- |
| `SIGN_UNKNOWN` | No attempt yet |
| `SIGN_SUCCESS` | Sign server accepted the credentials |
| `SIGN_EFAILED` | Could not connect to authentication server |
| `SIGN_EILLEGAL` | Authentication cancelled due to wrong input |
| `SIGN_EALERT` | Authentication server processing error |
| `SIGN_EALERT_COOP` | COG account not linked, or cannot log in to the selected server |
| `SIGN_EABORT` | Authentication server internal process crash |
| `SIGN_ERESPONSE` | Abnormal authentication response |
| `SIGN_EDATABASE` | Database access failure |
| `SIGN_EPASS` | Wrong ID or password |
| `SIGN_ESUSPEND` | Account temporarily suspended |
| `SIGN_EELIMINATE` | Account permanently suspended |
| `SIGN_ECLOSE` | Service closed |
| `SIGN_ECLOSE_EX` | Too much login traffic — wait and retry |
| `SIGN_ENOTREADY` | Server not ready |
| `SIGN_EALREADY` | Already logged in elsewhere |
| `SIGN_EIPADDR` | IP address / region blocked |
| `SIGN_ERIGHT` | Insufficient account rights |
| `SIGN_EAPP` | Unexpected launcher error |
| `SIGN_EOTHER` | ID authentication failed (catch-all) |
| `SIGN_ECOGLINK` | COG account not linked to PSN (console only) |
| `SIGN_ECOGCODE` | COG code error (console only) |
| `SIGN_ETOKEN` | Token error (console only) |
| `SIGN_EMAINTE` | Server maintenance (console only) |

---

### getUserId / getPassword

Returns the username / password used in the last `loginCog` call. Useful for re-authenticating without prompting the user again.

```js
var username = window.external.getUserId();
var password = window.external.getPassword();
```

---

### getServerListXml

Returns the raw server-list XML. See [Server List XML](#server-list-xml) for the full structure.

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

### getAccountRights

Returns XML listing the rights attached to the account. See [Account Rights](#account-rights).

```js
var rights = window.external.getAccountRights();
```

---

### getIniLastServerIndex / setIniLastServerIndex

Read and write the index of the server last selected by the user. The value is stored in the game's INI file and persists across sessions.

```js
var idx = window.external.getIniLastServerIndex();
window.external.setIniLastServerIndex(1);
```

---

### getMhfBootMode

Returns the current boot mode.

```js
var mode = window.external.getMhfBootMode();
```

| Value | Meaning |
| ----- | ------- |
| `_MHF_NORMAL` | Normal launch |
| `_MHF_SELFUP` | Self-update mode |
| `_MHF_DMM_SELF_UPDATE` | DMM self-update mode |
| `_MHF_AUTOLC` | Auto-launch: skip login UI and go directly to character select |
| `_MHF_DMM_AUTO_LAUNCH` | DMM auto-launch |

`_MHF_AUTOLC` and `_MHF_DMM_AUTO_LAUNCH` trigger automatic `loginCog` / `loginDmm` if credentials are cached.

---

### getMhfMutexNumber

Returns the process mutex number. Used to namespace cookie keys (`cogid{mutex}`, `pw{mutex}`) so multiple game instances do not share credentials.

```js
var mutex = window.external.getMhfMutexNumber();
```

---

### getLauncherReturnCode

Returns the launcher exit code.

| Value | Meaning |
| ----- | ------- |
| `'NULL'` | No code set |
| `'NORMAL'` | Normal exit |
| `'SELFUP'` | Self-update completed |
| `'ERR'` | Error exit |

---

### isEnableSessionId

Returns `true` if the session is valid. Returns `false` during server maintenance. Called after the update step completes — if `false`, a maintenance dialog is shown instead of the character selector.

---

### startUpdate

Checks for available updates and starts the download if one is found. Returns `false` if the `updateDisabled` flag is set in the launcher JS, in which case the update step is skipped entirely and the launcher proceeds directly to the character selector. Track progress with [`getUpdateStatus`](#getUpdateStatus) and [`getUpdatePercentageTotal / getUpdatePercentageFile`](#getUpdatePercentageTotal--getUpdatePercentageFile).

The `updateDisabled` flag is a top-level variable in the launcher JavaScript (`var updateDisabled = true`). Private server builds typically keep it `true` to skip patching.

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
| `'UM_UPDATE_NG'` | Update failed |

---

### getUpdatePercentageTotal / getUpdatePercentageFile

Return the overall update progress and the current-file progress, both as integers 0–100. Polled every ~50 ms to animate the progress bars.

```js
var total = window.external.getUpdatePercentageTotal();
var file  = window.external.getUpdatePercentageFile();
```

---

### extractLog

Extracts internal log messages from the game process. Polled every ~100 ms and displayed in the launcher's debug log panel.
