# Launcher

The launcher supports Javascript 1.3 version

### Launcher Internal functions

| FUNCTION                 | DESCRIPTION                                      |
| ------------------------ | ------------------------------------------------ |
| [playSound](#playSound)  | plays a song that is already inside the launcher |
| getMhfBootMode           |                                                  |
| getIniLastServerIndex    |                                                  |
| setIniLastServerIndex    |                                                  |
| getServerListXml         |                                                  |
| getMhfMutexNumber        |                                                  |
| minimizeWindow           |                                                  |
| closeWindow              |                                                  |
| openMhlConfig            |                                                  |
| restartMhf               |                                                  |
| openBrowser              |                                                  |
| beginDrag                |                                                  |
| getUserId                |                                                  |
| getPassword              |                                                  |
| loginCog                 |                                                  |
| loginHangame             |                                                  |
| loginDmm                 |                                                  |
| getLastAuthResult        |                                                  |
| getSignResult            |                                                  |
| startUpdate              |                                                  |
| getUpdatePercentageTotal |                                                  |
| getUpdatePercentageFile  |                                                  |
| getUpdateStatus          |                                                  |
| getAccountRights         |                                                  |
| getLauncherReturnCode    |                                                  |
| exitLauncher             |                                                  |
| isEnableSessionId        |                                                  |
| getCharacterInfo         |                                                  |
| deleteCharacter          |                                                  |
| selectCharacter          |                                                  |
| extractLog               |                                                  |
| debugGetIniUserId        |                                                  |
| debugGetIniPassword      |                                                  |

###### Typescript Type

```ts
declare module global {
  interface Window {
    external: LauncherFunctions;
  }
}

interface LauncherFunctions {
  playSound(song: LauncherSongs): void;
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
