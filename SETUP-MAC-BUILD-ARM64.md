# Build Doctipro Assist sur Mac Apple Silicon — séquence copiable

Ciblé **arm64 uniquement** (Mac M-series). Versions tirées du job `build-for-macOS`
de `.github/workflows/flutter-build.yml`, ligne par ligne — rien de deviné.
Variante Intel et détails : voir `SETUP-MAC-BUILD.md`.

> ⚠️ **PRÉALABLE OBLIGATOIRE côté Windows** : le fork doit être poussé.
> `origin/doctipro-assist` était encore au commit « build 1 » (0882d354) ;
> les 12 commits de branding (dont TOUT le macOS) sont locaux.
> Sur le PC : `cd C:\Users\Ben\Projects\doctipro-assist && git push origin doctipro-assist`
> Le submodule `hbb_common-assist` est déjà à jour sur GitHub (437906a1).

---

## 1. Prérequis (une seule fois, ~20 min)

```bash
xcode-select --install
brew install llvm create-dmg pkg-config

# NASM 2.16.03 depuis le site officiel — surtout PAS `brew install nasm` (met du 3.x, casse ffmpeg/aom)
cd /tmp
curl -LO https://www.nasm.us/pub/nasm/releasebuilds/2.16.03/macosx/nasm-2.16.03-macosx.zip
unzip -o nasm-2.16.03-macosx.zip
sudo cp nasm-2.16.03/nasm /usr/local/bin/nasm
nasm --version          # DOIT afficher 2.16.03

# Rust 1.81 (pas 1.75 : cidre l'exige sur macOS)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup toolchain install 1.81 --component rustfmt --target aarch64-apple-darwin

# vcpkg au commit épinglé du CI
git clone https://github.com/microsoft/vcpkg ~/vcpkg
cd ~/vcpkg && git checkout 120deac3062162151622ca4860575a33844ba10b && ./bootstrap-vcpkg.sh -disableMetrics
echo 'export VCPKG_ROOT=~/vcpkg' >> ~/.zshrc && export VCPKG_ROOT=~/vcpkg

# Flutter 3.24.5 stable arm64
cd ~ && curl -LO https://storage.googleapis.com/flutter_infra_release/releases/stable/macos/flutter_macos_arm64_3.24.5-stable.zip
unzip -q flutter_macos_arm64_3.24.5-stable.zip
echo 'export PATH=~/flutter/bin:$PATH' >> ~/.zshrc && export PATH=~/flutter/bin:$PATH
flutter --disable-analytics && flutter precache --macos
```

## 2. Cloner le fork + submodule + bridge FRB

```bash
cd ~ && git clone -b doctipro-assist https://github.com/Doctipro/doctipro-assist.git
cd doctipro-assist
git submodule update --init --recursive
rustup override set 1.81

# Fichiers bridge générés par le CI (gitignorés). Artefact vérifié vivant le 22/09, expire le 16/11/2026.
gh run download 32195903350 -R Doctipro/doctipro-assist -n bridge-artifact -D /tmp/bridge
cp /tmp/bridge/src/bridge_generated*.rs src/
cp /tmp/bridge/flutter/lib/generated_bridge* flutter/lib/
cp /tmp/bridge/flutter/macos/Runner/bridge_generated.h flutter/macos/Runner/
cp /tmp/bridge/flutter/ios/Runner/bridge_generated.h flutter/ios/Runner/
```

## 3. Patcher le SDK Flutter (les 2 étapes du CI)

```bash
cp ~/doctipro-assist/.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff ~/flutter/
cd ~/flutter && git apply flutter_3.24.4_dropdown_menu_enableFilter.diff
sed -i '' -e 's/_setFramesEnabledState(false);/\/\/_setFramesEnabledState(false);/g' \
  packages/flutter/lib/src/scheduler/binding.dart
grep -n '_setFramesEnabledState(false);' packages/flutter/lib/src/scheduler/binding.dart  # doit être commenté
```

## 4. Libs natives vcpkg (30-90 min la 1re fois)

```bash
cd ~/doctipro-assist
VCPKG_DEFAULT_HOST_TRIPLET=arm64-osx $VCPKG_ROOT/vcpkg install \
  --triplet arm64-osx --x-install-root="$VCPKG_ROOT/installed"
```

## 5. Build (min macOS 12.3 — les 4 sed du CI, spécifiques arm64)

```bash
cd ~/doctipro-assist
MIN=12.3
sed -i '' -e "s/MACOSX_DEPLOYMENT_TARGET\=[0-9]*.[0-9]*/MACOSX_DEPLOYMENT_TARGET=${MIN}/" build.py
sed -i '' -e "s/platform :osx, '.*'/platform :osx, '${MIN}'/" flutter/macos/Podfile
sed -i '' -e "s/osx_minimum_system_version = \"[0-9]*.[0-9]*\"/osx_minimum_system_version = \"${MIN}\"/" Cargo.toml
sed -i '' -e "s/MACOSX_DEPLOYMENT_TARGET = [0-9]*.[0-9]*;/MACOSX_DEPLOYMENT_TARGET = ${MIN};/" flutter/macos/Runner.xcodeproj/project.pbxproj

./build.py --flutter --hwcodec --unix-file-copy-paste --screencapturekit
```

**Sortie** : `./flutter/build/macos/Build/Products/Release/DoctiproAssist.app`

Test immédiat sans signature (clic droit → Ouvrir, 2×) :
```bash
xattr -dr com.apple.quarantine ./flutter/build/macos/Build/Products/Release/DoctiproAssist.app
open ./flutter/build/macos/Build/Products/Release/DoctiproAssist.app
```
Champs ID/Relay vides dans l'UI = **normal** (serveur + clé compilés dans le binaire).

## 6. Signature + notarisation (compte Apple Developer — actif depuis le 22/09)

Préparer une fois :
- Xcode → Settings → Accounts → ajouter l'Apple ID → **Manage Certificates → + → Developer ID Application**.
- Team ID : Apple Developer → Membership.
- Mot de passe d'application : appleid.apple.com → Connexion et sécurité → Mots de passe des apps.
- Pas besoin de créer l'App ID : hors App Store, le certificat Developer ID suffit.

```bash
cd ~/doctipro-assist/flutter/build/macos/Build/Products/Release
security find-identity -v -p codesigning        # récupérer le nom exact de l'identité

APP="DoctiproAssist.app"
IDENTITY="Developer ID Application: <Nom> (<TEAMID>)"

codesign --force --options runtime --deep --strict -s "$IDENTITY" "$APP" -vvv
codesign --verify --deep --strict --verbose=2 "$APP"

# Notarisation via l'Apple ID (plus simple que le rcodesign du CI)
ditto -c -k --keepParent "$APP" DoctiproAssist.zip
xcrun notarytool submit DoctiproAssist.zip \
  --apple-id <ton-email-apple> --team-id <TEAMID> --password <mdp-application> --wait
xcrun stapler staple "$APP"
spctl -a -vvv -t install "$APP"                 # attendu : accepted / source=Notarized Developer ID
```

DMG distribuable (signer l'app AVANT, puis signer + notariser le dmg) :
```bash
cd ~/doctipro-assist
create-dmg --icon "DoctiproAssist.app" 200 190 --hide-extension "DoctiproAssist.app" \
  --window-size 800 400 --app-drop-link 600 185 DoctiproAssist-1.4.9-arm64.dmg \
  ./flutter/build/macos/Build/Products/Release/DoctiproAssist.app
codesign --force --options runtime --deep --strict -s "$IDENTITY" DoctiproAssist-1.4.9-arm64.dmg
xcrun notarytool submit DoctiproAssist-1.4.9-arm64.dmg \
  --apple-id <email> --team-id <TEAMID> --password <mdp-app> --wait
xcrun stapler staple DoctiproAssist-1.4.9-arm64.dmg
```

## 7. Avant tout test de prise en main réelle

Purger les enregistrements périmés côté hbbs dev3, sinon l'avertissement E2E
« Connexion non sécurisée / pk mismatch » réapparaît (diagnostic du 21/08) — sur le serveur, en root :
```bash
docker stop rustdesk-hbbs
sqlite3 /home/doctibe/rustdesk/data/db_v2.sqlite3 "DELETE FROM peer;"
docker start rustdesk-hbbs
```
⚠️ base en WAL : éditer le vrai fichier, jamais une copie.
Le poste de contrôle doit lui aussi tourner sur Doctipro Assist (même clé en dur).

## Pièges déjà rencontrés (ne pas les re-découvrir)
- `brew install nasm` → NASM 3.x → build ffmpeg/aom cassé. Toujours le 2.16.03 officiel.
- Rust 1.75 (version Windows) → échec cidre sur macOS. C'est **1.81** ici.
- Pas de moteur Flutter custom sur Mac (c'est une spécificité Windows x64).
- Si la fenêtre principale ne s'affiche pas au 1er lancement : suspecter
  `customModule="RustDesk"` dans `MainMenu.xib` (résolu au build via customModuleProvider="target").
- « not ready » persistant = process client coincé → tout tuer et relancer, pas réinstaller.
