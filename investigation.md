# Investigation: GPG check not working on Windows runner

- **Reported:** https://community.sonarsource.com/t/sonarqube-scan-action-gpg-check-not-working-on-windows-runner/186785
- **Affected version:** `SonarSource/sonarqube-scan-action@v8.2.1`
- **Environment:** Windows runner with native GnuPG (Gpg4win) at `C:\Program Files (x86)\GnuPG\bin\gpg.exe`
- **Verdict:** ✅ **Confirmed bug** in `src/main/gpg-verification.js`

## Summary

On Windows, the action converts every path it hands to `gpg` (the `--homedir`, the
signature file, and the ZIP file) into an **MSYS/Git-Bash-style path** such as
`/r/GHEC/Build_B1/_temp/gpg-957091c8`. That format is only understood by the
**Git-for-Windows** build of GPG. When the `gpg` on `PATH` is the **native GnuPG
(Gpg4win)** — as in the reporter's environment — those paths are meaningless, so GPG
cannot create/read its home directory, keyring, sockets, or lock files, and signature
verification fails.

## Root cause

`convertToUnixPath()` in `src/main/gpg-verification.js:106-118`:

```js
export function convertToUnixPath(windowsPath) {
  if (process.platform !== "win32") {
    return windowsPath;
  }
  let unixPath = windowsPath.replaceAll('\\', "/");
  unixPath = unixPath.replace(/^([A-Za-z]):/, (match, drive) => {
    return `/${drive.toLowerCase()}`;   // R:\... -> /r/...
  });
  return unixPath;
}
```

For the reporter's temp dir `R:\GHEC\Build_B1\_temp\gpg-957091c8` this produces:

```
R:\GHEC\Build_B1\_temp\gpg-957091c8   →   /r/GHEC/Build_B1/_temp/gpg-957091c8
```

which is exactly the path seen in the failure logs.

The function's own comment states the assumption:

> `GPG on Windows (from Git for Windows) expects Unix-style paths` — `src/main/gpg-verification.js:102`

That assumption is wrong for the common case. There are two GPG builds on Windows:

| GPG build | Understands `/r/GHEC/...` (MSYS) | Understands `R:/GHEC/...` (native) |
|---|---|---|
| Git for Windows / MSYS2 `gpg` | ✅ | ✅ |
| **Native GnuPG / Gpg4win** (`C:\Program Files (x86)\GnuPG\bin\gpg.exe`) | ❌ | ✅ |

The MSYS drive-letter form (`/r/...`) is understood by *only one* of the two builds,
whereas the plain Windows form with forward slashes (`R:/...`) is understood by **both**.
By rewriting `R:` → `/r`, the action picks the form that breaks native GnuPG.

## How the converted path is used

All three call sites feed native GnuPG a path it cannot resolve:

- `src/main/gpg-verification.js:165` — `--homedir` for `--recv-keys` (key import)
- `src/main/gpg-verification.js:247` — `--homedir` for `--verify`
- `src/main/gpg-verification.js:250-251` — the `.asc` signature path and the ZIP path

## Mapping to the reported errors

With `--homedir /r/GHEC/Build_B1/_temp/gpg-957091c8`, native GnuPG treats the argument
as a bogus location and cannot create/open its home dir. That directly explains every
symptom in the report:

| Reported symptom | Explanation |
|---|---|
| `pubring.kbx` → *No such file or directory* | Keyring cannot be created/read under the bogus homedir |
| `can't create '…/gnupg_spawn_dirmngr_sentinel.lock'` → *The system cannot find the path specified* | Native Windows API cannot resolve `/r/...`; parent path does not exist |
| `can't connect to the dirmngr: No such file or directory` | `dirmngr` socket cannot be created under the bogus homedir |
| `Kein Dirmngr` (keyserver retrieval failed) | Consequence of dirmngr never starting → `--recv-keys` fails |

The reporter's own suggested fix — use `R:/GHEC/Build_B1/_temp/gpg-957091c8` — matches
the analysis above.

## Why it isn't caught by CI / existing tests

- The unit tests in `src/main/__tests__/gpg-verification.test.js:132-163` only assert the
  transformation itself (`C:\a\_temp\gpg-home` → `/c/a/_temp/gpg-home`). They **encode the
  buggy behaviour as the expected behaviour**, so they pass while the real bug persists.
- GitHub-hosted `windows-latest` runners resolve `gpg` to the **Git-for-Windows** build,
  which happens to accept `/r/...`. The bug only surfaces when the native GnuPG (Gpg4win)
  is first on `PATH` — the reporter's self-hosted setup. This is why it escaped testing.

## Suggested fix (not applied — investigation only)

On Windows, do **not** rewrite the drive letter. Normalise separators only, keeping a
native Windows path with forward slashes, which both GPG builds accept:

```js
export function convertToUnixPath(windowsPath) {
  if (process.platform !== "win32") {
    return windowsPath;
  }
  // Keep the drive letter (R:/...). Native GnuPG (Gpg4win) cannot resolve the
  // MSYS "/r/..." form, and Git-for-Windows GPG understands "R:/..." too.
  return windowsPath.replaceAll("\\", "/");
}
```

The corresponding tests at `src/main/__tests__/gpg-verification.test.js:132-153` should be
updated to expect `C:/a/_temp/gpg-home` (and the function likely renamed, since it no
longer produces a Unix path).

## Secondary observation (not the reported bug)

`src/main/run-sonar-scanner.js:147` invokes `${scannerDir}/jre/bin/java` without a `.exe`
suffix and with forward slashes. This only runs when `SONAR_ROOT_CERT` is set (truststore
handling) and is unrelated to the GPG failure, but is worth a separate look for Windows
robustness.

## Reproducer

A CI reproducer is provided in `.github/workflows/issue-reproduced.yml`. It runs on
`windows-latest`, installs the **native GnuPG (Gpg4win)** and puts it first on `PATH` to
match the reporter's environment, then runs the action (`uses: ./`, which executes the
committed `dist/index.js`). It expects the action to **fail at GPG signature
verification**. It also includes a direct step showing native `gpg` rejecting an
MSYS-style `--homedir`, isolating the root cause. The workflow triggers `on: push`.
