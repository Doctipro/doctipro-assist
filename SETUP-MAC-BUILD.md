# SETUP — Build macOS de Doctipro Assist

Marche à suivre turnkey sur un Mac, calquée sur le job macOS de
`.github/workflows/flutter-build.yml` (versions EXACTES du CI, pas devinées).
Le branding macOS est déjà commité — voir « État du branding » en bas.

## Versions (source de vérité = flutter-build.yml)

| Composant | Version | Note |
|---|---|---|
| Rust | **1.81** (`MAC_RUST_VERSION` — PAS 1.75 comme Windows, requis par cidre) | + `rustfmt`, target `aarch64-apple-darwin` (Apple Silicon) ou `x86_64-apple-darwin` (Intel) |
| Flutter | **3.24.5 stable** | + patch dropdown + workaround binding.dart (voir §3) ; PAS de moteur custom sur macOS (spécifique Windows x64) |
| LLVM | `brew install llvm` | le CI prend le llvm de brew sur mac |
| NASM | **2.16.x depuis le site officiel** — ⚠️ PAS `brew install nasm` (installe 3.x, casse le build ffmpeg) | |
| vcpkg | commit **120deac3062162151622ca4860575a33844ba10b** | triplet `arm64-osx` (Apple Silicon) / `x64-osx` (Intel) |
| Xcode | Xcode + Command Line Tools récents (le CI tourne sur macos-14/15) | `xcode-select --install` |
| Autres | `brew install create-dmg pkg-config` ; Python 3 | |

## 1. Prérequis (une fois)

```bash
xcode-select --install                      # CLT (ou Xcode complet depuis l'App Store)
brew install llvm create-dmg pkg-config nasm # ⚠️ puis REMPLACER nasm par le 2.16.x officiel si brew a mis 3.x :
nasm --version                               # doit afficher 2.16.x — sinon: https://www.nasm.us/pub/nasm/releasebuilds/2.16.03/macosx/
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup toolchain install 1.81 --component rustfmt
git clone https://github.com/microsoft/vcpkg ~/vcpkg && cd ~/vcpkg && git checkout 120deac3062162151622ca4860575a33844ba10b && ./bootstrap-vcpkg.sh -disableMetrics
export VCPKG_ROOT=~/vcpkg                    # à mettre dans ~/.zshrc
# Flutter 3.24.5
cd ~ && curl -LO https://storage.googleapis.com/flutter_infra_release/releases/stable/macos/flutter_macos_arm64_3.24.5-stable.zip   # (arm64 ; Intel: flutter_macos_3.24.5-stable.zip)
unzip -q flutter_macos_arm64_3.24.5-stable.zip && export PATH=~/flutter/bin:$PATH
flutter --disable-analytics && flutter precache --macos
```

## 2. Cloner le fork + submodule + bridge

```bash
git clone -b doctipro-assist git@github.com:Doctipro/doctipro-assist.git && cd doctipro-assist
git submodule update --init --recursive       # hbb_common-assist (serveurs+clé doctipro.be)
rustup override set 1.81                       # override local Mac (le repo Windows utilise 1.75)
# Fichiers bridge FRB (générés par le CI, gitignorés) — les récupérer du run Build 1 :
gh run download 32195903350 -n bridge-artifact -D /tmp/bridge
cp /tmp/bridge/src/bridge_generated*.rs src/
cp /tmp/bridge/flutter/lib/generated_bridge* flutter/lib/
cp /tmp/bridge/flutter/macos/Runner/bridge_generated.h flutter/macos/Runner/
cp /tmp/bridge/flutter/ios/Runner/bridge_generated.h flutter/ios/Runner/
```

## 3. Patcher le SDK Flutter (comme le CI)

```bash
cp .github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff ~/flutter/
cd ~/flutter && git apply flutter_3.24.4_dropdown_menu_enableFilter.diff
# Workaround flutter#133533 (étape « Workaround for flutter issue » du CI) :
sed -i '' -e 's/_setFramesEnabledState(false);/\/\/_setFramesEnabledState(false);/g' packages/flutter/lib/src/scheduler/binding.dart
```

## 4. Libs natives vcpkg (long la 1re fois, ~30-90 min)

```bash
cd ~/doctipro-assist
$VCPKG_ROOT/vcpkg install --x-install-root=$VCPKG_ROOT/installed   # lit vcpkg.json (triplet auto)
```

## 5. Build (Apple Silicon — pour Intel, sauter les 4 sed)

```bash
# arm64 uniquement : min macOS 12.3 (séquence exacte du CI)
MIN=12.3
sed -i '' -e "s/MACOSX_DEPLOYMENT_TARGET\=[0-9]*.[0-9]*/MACOSX_DEPLOYMENT_TARGET=${MIN}/" build.py
sed -i '' -e "s/platform :osx, '.*'/platform :osx, '${MIN}'/" flutter/macos/Podfile
sed -i '' -e "s/osx_minimum_system_version = \"[0-9]*.[0-9]*\"/osx_minimum_system_version = \"${MIN}\"/" Cargo.toml
sed -i '' -e "s/MACOSX_DEPLOYMENT_TARGET = [0-9]*.[0-9]*;/MACOSX_DEPLOYMENT_TARGET = ${MIN};/" flutter/macos/Runner.xcodeproj/project.pbxproj

./build.py --flutter --hwcodec --unix-file-copy-paste --screencapturekit   # (Intel : sans --screencapturekit)
```
**Sortie** : `./flutter/build/macos/Build/Products/Release/DoctiproAssist.app`

DMG (optionnel, comme le CI) :
```bash
create-dmg --icon "DoctiproAssist.app" 200 190 --hide-extension "DoctiproAssist.app" \
  --window-size 800 400 --app-drop-link 600 185 DoctiproAssist-1.4.9.dmg \
  ./flutter/build/macos/Build/Products/Release/DoctiproAssist.app
```

## 6. Signature / notarisation (compte Apple Developer 99 $/an)

Sans signature : l'app se lance via **clic droit → Ouvrir** (2×) ou
`xattr -dr com.apple.quarantine DoctiproAssist.app`. OK pour tester, pas pour distribuer.

Avec le compte (une fois l'adhésion active) :
1. Xcode → Settings → Accounts → ajouter l'Apple ID → créer un certificat **Developer ID Application**.
2. Signer : `codesign --force --options runtime --deep --strict -s "Developer ID Application: <Nom> (<TEAMID>)" DoctiproAssist.app`
3. Notariser : zipper l'app puis
   `xcrun notarytool submit DoctiproAssist.zip --apple-id <email> --team-id <TEAMID> --password <app-specific-pwd> --wait`
4. Agrafer : `xcrun stapler staple DoctiproAssist.app` — l'app s'ouvre alors sans avertissement.
   (Pour un .dmg : signer l'app AVANT create-dmg, puis notariser/stapler le dmg.)
- Bundle id déjà en place : **`lu.doctipro.assist`** (pbxproj + xcconfig) — créer l'App ID correspondant n'est pas nécessaire pour Developer ID (hors App Store), le certificat suffit.

## État du branding macOS (commité)
- `AppInfo.xcconfig` : PRODUCT_NAME=DoctiproAssist, bundle id `lu.doctipro.assist`, copyright Doctipro.
- `Info.plist` : CFBundleDisplayName « Doctipro Assist », URL scheme **`doctiproassist`** (deep link), URLName `lu.doctipro.assist`.
- `AppIcon.icns` : régénéré avec le D teal Doctipro (8 tailles 32→1024, conteneur ICNS/PNG).
- `res/mac-icon.png` (1024) + `res/mac-tray-dark-x2.png`/`light-x2` (glyphe D template) : régénérés.
- pbxproj/xcscheme : produit `DoctiproAssist.app` ; CI et build.py adaptés.
- Services launchd/scripts : templates `com.carriez.rustdesk`/« RustDesk » réécrits AU RUNTIME par `correct_app_name()` (src/platform/macos.rs:305) avec le bundle id réel + APP_NAME — rien à changer.
- Restes non visibles assumés : NSLog «[RustDesk]» dans MainFlutterWindow.swift (Console.app uniquement), customModule="RustDesk" dans MainMenu.xib (customModuleProvider="target" → résolu vers le module du target au build ; à surveiller au 1er lancement : si la fenêtre principale ne s'affiche pas, c'est ce point).

## ⚠️ À savoir avant le 1er build
- Le rendez-vous/relay/clé doctipro.be sont compilés (submodule) — champs UI vides = normal.
- Avant tout test de prise en main : purger les enregistrements périmés côté hbbs (cf. diagnostic « pk mismatch » du 21/08) sinon le même avertissement E2E apparaîtra.
