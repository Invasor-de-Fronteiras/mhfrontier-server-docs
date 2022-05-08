# Launcher

### Launcher Internal functions

| FUNCTION  | DESCRIPTION                                      |
| --------- | ------------------------------------------------ |
| playSound | plays a song that is already inside the launcher |

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
