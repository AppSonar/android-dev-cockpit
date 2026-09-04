# AGENTS.md – Regeln für KI-Assistenten

## Das Gesamtbild

```
<projekte>\
├── android-dev-cockpit\   ← dieses Repo: die Werkzeuge, generisch
├── <app-repo-1>\          ← ein App-Repo: der Code
└── <app-repo-2>\          ← beliebig viele weitere, werden selbst gefunden
```

App-Repos liegen als Geschwister neben diesem Repo, nie darin; der Ordnername
dieses Repos ist frei.

## Wo was hingehört

| Was | Wohin |
| --- | --- |
| Toolchain (JDK, Android SDK, Gradle-Cache) | `toolchain/` (gitignoriert) |
| Build- und Installationsskripte | `scripts/`, `install.ps1` (versioniert) |
| Fertige APKs, Build-Ergebnisse | `builds/` (gitignoriert) |
| Planungen, Notizen, Mitschriften | `planning/<projekt>/` (gitignoriert) |
| App-Quellcode | das jeweilige App-Repo |

## Regeln

- Dieses Repo ist generisch und öffentlich: keine App-Namen, -Listen oder
  -Pfade, auch nicht in Kommentaren; Projektwissen gehört ins App-Repo.
- Nichts dauerhaft am System verändern: kein `winget install`, kein `setx`,
  keine Installer; Umgebungsvariablen setzt `scripts/_env.ps1` pro Shell.
- Secrets nur in die gitignorierte `.env`; `.env.example` nennt die Schlüssel
  ohne Werte.
- `app-development.code-workspace` und `package.json` sind generiert und
  werden nie von Hand editiert: neues Skript in `scripts/`, Generator anpassen.
- PowerShell-Skripte sind UTF-8 mit BOM.
- Fehler im Klartext mit Handlungsanweisung über `Stop-WithHint`, Ausgaben
  über `Write-Step`/`Write-Ok`/`Write-Info`/`Write-Warn2`/`Write-Fail` aus
  `_env.ps1`. Parameter wie `-App <id>` setzen die generierten Tasks, nicht
  der Benutzer.
- npm-Skripte rufen `pwsh` aus dem PATH mit relativen Skriptpfaden auf, nie
  den vollen Pfad; Variablen vor einem Doppelpunkt als `${id}:push` schreiben.

## Die Umgebung

- `install.cmd` → `install.ps1`, idempotent.
- `scripts/_env.ps1` lädt die `.env` und setzt `JAVA_HOME`, `ANDROID_HOME`,
  `GRADLE_USER_HOME` und `PATH` für die laufende Shell (Dot-Sourcing).
- `scripts/discover-apps.ps1` findet Apps unterhalb von `PROJECTS_ROOT` selbst
  (Verzeichnis mit `gradlew.bat` und `app/`-Modul); es gibt keine App-Liste.
- Repos ohne baubare App kommen über `WORKSPACE_EXTRA_DIRS` in der `.env` in
  den Workspace (kommagetrennt, relativ zu `PROJECTS_ROOT`).
- Archiv-Kopien eines App-Ordners ergeben dieselbe App-Id und der erste
  Treffer gewinnt (`(Get-AppOrFail -Id '<id>').Path`); Kopien umbenennen oder
  deren `gradlew.bat` entfernen.

## Was ein App-Repo mitbringen muss

- Ein Verzeichnis mit `gradlew.bat`, darin ein Modul `app/` mit
  `build.gradle.kts` und `applicationId`.
- Anzeigename aus `rootProject.name` in `settings.gradle.kts`, sonst der
  Ordnername; Single-App-Repo und Monorepo (`<repo>/apps/<app>/gradlew.bat`)
  werden beide gefunden.
- Toolchain: JDK 17 und SDK-Platform 34; Abweichungen über `ANDROID_PLATFORM`
  und `ANDROID_BUILD_TOOLS` in der `.env`, nie im Skript.

## Für jedes App-Repo gilt

- npm-Skripte des Cockpits: `<app>`, `<app>:test`, `<app>:push`,
  `<app>:reinstall`, `<app>:logs`, `<app>:clean`, `<app>:uninstall`.
- Release-Weg: Push auf `main` → Unit-Tests → Debug- und Release-APK →
  GitHub-Release `latest` mit `<kurzname>-version.json`; bei releasewirksamen
  Änderungen `versionCode` erhöhen.
- Gemeinsamer Release-Keystore; Secrets `RELEASE_KEYSTORE` und
  `RELEASE_STORE_PASSWORD` je Repo.
- Nach Änderungen an der gemeinsamen Bibliothek alle App-Builds von Hand
  anstoßen: `gh workflow run build.yml -R <org>/<app>`.

## Testgerät (Xiaomi/MIUI)

- Das Gerät lässt sich nicht per ADB aufwecken; Wachzustand prüfen mit
  `adb shell dumpsys power | grep mWakefulness`.
- Jede Neuinstallation über ADB wird mit `INSTALL_FAILED_USER_RESTRICTED`
  abgewiesen, einen ADB-Workaround gibt es nicht. `<app>:push` legt die APK in
  die Downloads des Handys, der Nutzer installiert sie einmal von Hand. Danach
  läuft `<app>` (`installDebug`) automatisch durch.
- `adb shell input tap/swipe/keyevent` und `am start` auf nicht-exportierte
  Activities sind gesperrt. Getestet wird beobachtend: der Mensch bedient, der
  Agent liest `adb exec-out screencap -p > shot.png` und `<app>:logs`.
  Screenshots des Nutzers liegen unter `/sdcard/DCIM/Screenshots/`.
- Debug- und Release-Signatur derselben App nicht auf einem Gerät mischen
  (`INSTALL_FAILED_UPDATE_INCOMPATIBLE`); die andere Version vorher entfernen.
- Ein Release-Keystore wird im Debug-Loop nie gebraucht und gehört nicht in
  dieses Repo.
- Deinstallation des Debug-Builds löscht lokale Keys und Daten auf dem Gerät.
