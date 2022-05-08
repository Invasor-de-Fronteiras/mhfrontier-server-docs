# Launcher

The launcher supports Javascript 1.3 version

### Launcher Internal functions

All internal functions are inside `window.external`

Also trigger an error when poorly executedm you can get errors with try/catch:

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
