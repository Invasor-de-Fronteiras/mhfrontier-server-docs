# Launcher

When the launcher opens it consults two server, [srv-mhf.capcom-networks.jp](#launcher-hosts) and [cog-members.mhf-z.jp](#launcher-hosts), [srv-mhf.capcom-networks.jp](#launcher-hosts) to get the server list that is consumable by the launcher [`window.external.getServerListXml`](#getServerListXml)
and [cog-members.mhf-z.jp](#launcher-hosts) to get the html of launcher.

**The launcher supports Javascript 1.3 Version**

### Launcher Hosts

| NAME                                      | URI                        |
| ----------------------------------------- | -------------------------- |
| [Server information](#server-information) | srv-mhf.capcom-networks.jp |
| [Launcher](#launcher)                     | cog-members.mhf-z.jp       |

#### Server information

##### Routes

| METHOD | ROUTE                  | DESCRIPTION                                |
| ------ | ---------------------- | ------------------------------------------ |
| UNK    | /server/unique.php     | Check if the character name already exists |
| GET    | /server/serverlist.php | Returns server list                        |

###### /server/serverlist.php

Returns server list

Return Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server_groups>
   <group idx="0" nam="SERVER_NAME" ip="SERVER_IP" port="SERVER_PORT" />
</server_groups>
```

#### Launcher

##### Routes

| METHOD | ROUTE                | DESCRIPTION              |
| ------ | -------------------- | ------------------------ |
| GET    | /launcher/?ver=2.016 | Get Launcher HTML Source |

### Launcher Internal functions

All internal functions are inside `window.external`

Also trigger an error when poorly executed, you can get errors with try/catch:

```js
try {
  window.external.getLastAuthResult();
} catch (err) {
  // ...
}
```

| FUNCTION                          | DESCRIPTION                                      |
| --------------------------------- | ------------------------------------------------ |
| [playSound](#playSound)           | plays a song that is already inside the launcher |
| getMhfBootMode                    |                                                  |
| getIniLastServerIndex             |                                                  |
| setIniLastServerIndex             |                                                  |
| getServerListXml                  |                                                  |
| getMhfMutexNumber                 |                                                  |
| [minimizeWindow](#minimizeWindow) | minimize the window                              |
| [closeWindow](#closeWindow)       | close the window                                 |
| openMhlConfig                     |                                                  |
| [restartMhf](#restartMhf)         | restart the window                               |
| [openBrowser](#openBrowser)       | open an url in browser                           |
| [beginDrag](#beginDrag)           | frees the user to drag the window                |
| getUserId                         |                                                  |
| getPassword                       |                                                  |
| loginCog                          |                                                  |
| loginHangame                      |                                                  |
| loginDmm                          |                                                  |
| getLastAuthResult                 |                                                  |
| getSignResult                     |                                                  |
| startUpdate                       |                                                  |
| getUpdatePercentageTotal          |                                                  |
| getUpdatePercentageFile           |                                                  |
| getUpdateStatus                   |                                                  |
| getAccountRights                  |                                                  |
| getLauncherReturnCode             |                                                  |
| exitLauncher                      |                                                  |
| isEnableSessionId                 |                                                  |
| getCharacterInfo                  |                                                  |
| deleteCharacter                   |                                                  |
| selectCharacter                   |                                                  |
| extractLog                        |                                                  |
| debugGetIniUserId                 |                                                  |
| debugGetIniPassword               |                                                  |

###### Typescript Type

```ts
declare module global {
  interface Window {
    external: LauncherFunctions;
  }
}

interface LauncherFunctions {
  playSound(song: LauncherSongs): void;
  beginDrag(active: boolean): void;
  openBrowser(url: string): void;
  restartMhf(): void;
  minimizeWindow(): void;
  closeWindow(): void;
}

type LauncherSongs =
  | "IDR_WAV_SEL"
  | "IDR_WAV_OK"
  | "IDR_WAV_PRE_LOGIN"
  | "IDR_NIKU";
```

##### playSound

plays a song that is already inside the launcher

###### How to use:

```js
window.external.playSound(launcherSongType);
```

###### Launcher Songs Types

| ID                | Description                             |
| ----------------- | --------------------------------------- |
| IDR_WAV_SEL       | sound for when mouse hovers over things |
| IDR_WAV_OK        | sound for when actions are successful   |
| IDR_WAV_PRE_LOGIN | sound for when the user click in login  |
| IDR_NIKU          | sound for when login is successful      |

##### closeWindow

Close the Window

###### How to use:

```js
window.external.closeWindow();
```

##### beginDrag

frees the user to drag the window

###### How to use:

```js
window.external.beginDrag(boolean);
```

##### openBrowser

Open an url in Browser

###### How to use:

```js
window.external.openBrowser(url);
```

##### restartMhf

Restart the Window

###### How to use:

```js
window.external.restartMhf();
```

##### minimizeWindow

Minimize the Window

###### How to use:

```js
window.external.minimizeWindow();
```

##### closeWindow

Close the Window

###### How to use:

```js
window.external.closeWindow();
```
