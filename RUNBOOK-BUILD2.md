# RUNBOOK — Doctipro Assist « Build 2 » (branding groupé)

État au 21/08/2026. Base : branche `doctipro-assist` (fork RustDesk 1.4.9), Build 1 validé
(connexion serveur doctipro.be OK, run Actions 32195903350 : Windows x86_64 vert, seul le
job Linux armv7 sciter a échoué — sans impact).

## 1. Ce qui est DÉJÀ commité en local (à pousser par Ben)

- `dceccd63` — nom affiché « Doctipro Assist » (avec espace) dans TOUTES les chaînes
  traduites (`src/lang.rs` passe par le nouveau `get_display_name()` de `src/common.rs`),
  tooltip du tray (`src/tray.rs`), et masquage du lien « Powered by RustDesk »
  (`BUILTIN_SETTINGS hide-powered-by-me=Y` injecté dans `load_custom_client()`).
- `3ca02da7` — Flutter : teal Doctipro `#0A5560` (MyTheme.accent/accent50/accent80/button,
  colorScheme light ; dark = teal éclairci `#2E8C99` pour la lisibilité), titre du tabbar
  « Doctipro Assist », cartes About desktop + mobile : liens rustdesk.com → doctipro.lu,
  copyright Purslane → Doctipro, bandeau bleu → teal, lien privacy de la page
  d'installation → doctipro.lu.
- `9617828f` — plateformes : `Runner.rc` (CompanyName/LegalCopyright Doctipro),
  `main.cpp` (fallback titre `DoctiproAssist`), labels Android « Doctipro Assist »,
  macOS `PRODUCT_NAME = DoctiproAssist` (AppInfo.xcconfig + pbxproj + xcscheme) et
  adaptation de `.github/workflows/flutter-build.yml` + `build.py` aux chemins
  `DoctiproAssist.app` (sinon le job macOS casse).

Contraintes techniques assumées (vérifiées dans le code) :
- Le TITRE de la fenêtre principale reste `DoctiproAssist` (sans espace) : `main.cpp`
  retrouve l'instance existante par `FindWindowW(titre == APP_NAME)` pour le
  single-instance et le dispatch des deep links. Aucune mention RustDesk, mais pas
  d'espace possible ici.
- Le protocole devient automatiquement `doctiproassist://` (`get_uri_prefix()` =
  APP_NAME en minuscules) — rien d'autre à patcher côté client.
- À l'installation Windows, l'exe est renommé `DoctiproAssist.exe` automatiquement
  (`src/platform/windows.rs` copy_exe). L'artefact CI s'appelle toujours
  `rustdesk-...exe` : le renommer à la main avant distribution.
- Licence AGPLv3 : la mention `$license` reste dans l'About (obligatoire). Le code
  source du fork doit rester publiable si l'exe est distribué hors de Doctipro.

## 2. Ce qu'il me FAUT de Ben pour finir

### a) Asset logo (bloquant pour l'icône)
Fournir idéalement un carré 1024×1024 PNG fond transparent + déclinaisons. Fichiers à
remplacer (mêmes noms, mêmes emplacements) :
- **In-app (tabbar, accueil, tray runtime)** : `flutter/assets/icon.svg` (logo vectoriel
  principal) — et en option sans rebuild du SVG : `flutter/assets/logo.png`
  (+ `logo_light.png` / `logo_dark.png`, chargés en priorité s'ils existent).
- **Windows** : `flutter/windows/runner/resources/app_icon.ico` (icône exe/fenêtre,
  multi-tailles 16→256) ; `res/icon.ico` ; `res/tray-icon.ico` (tray).
- **macOS** : `flutter/macos/Runner/AppIcon.icns` ; `res/mac-icon.png` ;
  `res/mac-tray-dark-x2.png` / `res/mac-tray-light-x2.png` (template monochrome).
- **Linux/divers** : `res/icon.png` (1024), `res/32x32.png`, `res/64x64.png`,
  `res/128x128.png`, `res/128x128@2x.png`.
- **Android** (optionnel) : `flutter/android/app/src/main/res/mipmap-*/ic_launcher*.png`.
Une fois les fichiers déposés : `git add res flutter/assets flutter/windows/runner/resources
flutter/macos/Runner/AppIcon.icns flutter/android/app/src/main/res && git commit -m "build 2: icones Doctipro"`.

### b) Décision mode borne vs médecin
Non tranché → rien de codé. Si borne = réception seule, options candidates (BUILTIN_SETTINGS
dans `load_custom_client()`, même mécanique que hide-powered-by-me) à valider avant patch.

### c) Décision macOS
Build macOS possible tel quel (non signé → Gatekeeper « clic droit > Ouvrir »).
Signature/notarisation = compte Apple Developer 99 $/an (secrets
`MACOS_CODESIGN_IDENTITY` etc. déjà prévus dans le workflow).

## 3. Gestes manuels de Ben (dans l'ordre)

1. (après dépôt des icônes + commit) `cd C:\Users\Ben\Projects\doctipro-assist`
2. `git push origin doctipro-assist`
3. GitHub → Doctipro/doctipro-assist → Actions → « Flutter Nightly Build » →
   Run workflow → branche `doctipro-assist` (~30-45 min avec cache).
4. Récupérer l'artefact `rustdesk-unsigned-windows-x86_64` du run, renommer l'exe en
   `DoctiproAssist.exe`, tester : install, tray, About, connexion à un device,
   deep link `doctiproassist://connection/new/<ID>`.
5. **Après bascule du PC de Ben sur le client brandé** — patch Doctipro applicatif
   (repo doctipro, PAS fait, à faire à ce moment-là) :
   `app/Models/RemoteDevice.php:40` :
   `return 'rustdesk://connection/new/'.$this->device_id;`
   → `return 'doctiproassist://connection/new/'.$this->device_id;`
   (+ commentaires lignes RemoteDevice.php:10 et RemoteDeviceTable.php:16).
6. Promo préprod/prod (manuel) : déposer l'exe, 2 vars `.env`
   (RUSTDESK_HOST/RUSTDESK_KEY), exemption htaccess dev3.

## 4. État « prêt à builder » ?

- Prêt à builder SANS icône : oui — le build donnerait déjà un client 100 % sans mention
  « RustDesk » visible (nom, About, tray, couleurs, installeur), mais avec l'icône RustDesk.
- Pour respecter « tout grouper en un seul build » : attendre l'asset logo (2.a) et la
  décision mode borne (2.b) avant de dispatcher.
- Non couvert dans ce build : cosmétique #1443 (client_version, sysinfo, nettoyage des
  2 appareils en base) — côté Doctipro applicatif, hors périmètre fork.
