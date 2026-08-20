# SETUP — Build local Windows x86_64 de Doctipro Assist

Objectif : reproduire en local le job CI `build-for-windows-flutter` (x86_64) de
`.github/workflows/flutter-build.yml`, qui est la chaîne qui a produit le Build 1 validé.

## Versions (source de vérité = flutter-build.yml / bridge.yml, PAS devinées)

| Composant | Version CI | Où c'est défini |
|---|---|---|
| Rust | **1.75** (x86_64-pc-windows-msvc, + rustfmt) | `SCITER_RUST_VERSION` (utilisé aussi par le job flutter Windows) |
| Flutter | **3.24.5 stable x64** | `FLUTTER_VERSION` |
| Moteur Flutter | remplacé par `rustdesk/engine` `windows-x64-release.zip` | step « Replace engine » (x64 uniquement) |
| Patch Flutter SDK | `.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff` | step « Patch flutter » |
| LLVM/Clang | **15.0.6** | `LLVM_VERSION` |
| vcpkg | commit **120deac3062162151622ca4860575a33844ba10b** | `VCPKG_COMMIT_ID` |
| Triplet vcpkg | **x64-windows-static** (aussi en `VCPKG_DEFAULT_HOST_TRIPLET`) | matrix x86_64 |
| Python | 3.x (3.11.5 local OK) | `build.py` |
| Bridge FRB | générés par CI avec Flutter 3.22.3 + flutter_rust_bridge_codegen **1.80.1** | `bridge.yml` — PAS regénérés localement, récupérés de l'artefact CI |
| Features build | `--portable --flutter --skip-portable-pack --hwcodec --vram` | matrix x86_64 + step Build |

## Ce qui est INSTALLÉ / PRÉPARÉ (21/08/2026)

- Rust : rustup 1.29.0 (winget `Rustlang.Rustup`), toolchain **1.75.0** posée en
  `rustup override set 1.75` sur `C:\Users\Ben\Projects\doctipro-assist` (le default
  machine reste stable 1.98 pour ne pas polluer les autres projets). Vérifié :
  `rustc 1.75.0 (82e1608df)` dans le repo.
- vcpkg : cloné dans `C:\vcpkg`, checkout exact `120deac30…`, bootstrapé
  (`vcpkg version` OK). Poser `VCPKG_ROOT=C:\vcpkg`.
- Bridge FRB : les 6 fichiers générés récupérés de l'artefact `bridge-artifact` du run
  32195903350 (celui du Build 1) et posés : `src/bridge_generated.rs`,
  `src/bridge_generated.io.rs`, `flutter/lib/generated_bridge.dart`,
  `flutter/lib/generated_bridge.freezed.dart`, `flutter/{macos,ios}/Runner/bridge_generated.h`.
  (Fichiers gitignorés — à re-régénérer seulement si `src/flutter_ffi.rs` change.)
- Extras de packaging (gitignorés, `build-extras/`) : `WindowInjection.dll`
  (artefact `topmostwindow-artifacts-x64` du même run) + `usbmmidd_v2.zip`.
- Flutter **3.24.5 stable** extrait dans `C:\dev\flutter` (zip officiel
  `flutter_windows_3.24.5-stable.zip`, vérifié `flutter --version` = 3.24.5 / Dart 3.5.4),
  analytics désactivées, `flutter precache --windows` fait, **moteur remplacé** par
  `rustdesk/engine windows-x64-release.zip` dans
  `C:\dev\flutter\bin\cache\artifacts\engine\windows-x64-release\` (comme le CI),
  **patch dropdown appliqué** au SDK (`git apply flutter_3.24.4_dropdown_menu_enableFilter.diff`, OK).
  ⚠️ Ne pas lancer `flutter upgrade` dans ce SDK.
- LLVM/Clang **15.0.6** installé dans `C:\LLVM-15.0.6` (installeur officiel GitHub, /S),
  vérifié `clang --version`.
- **VS Build Tools 2022 17.14.39** + workload C++ (`--includeRecommended`) via winget :
  MSVC 14.44.35207, Windows SDK 10.0.26100, CMake VS intégré. `flutter doctor` : ✅
  Visual Studio détecté.
- `flutter pub get` exécuté dans `flutter/` : résolution OK sur 3.24.5 (aucun sed
  extended_text nécessaire — c'était uniquement pour le codegen 3.22.3).
- Variables d'env **utilisateur** posées (setx) : `VCPKG_ROOT=C:\vcpkg`,
  `LIBCLANG_PATH=C:\LLVM-15.0.6\bin`, `VCPKG_DEFAULT_HOST_TRIPLET=x64-windows-static`.
  (Ouvrir un NOUVEAU terminal pour les voir.)

## ⚠️ BLOQUANT restant — geste manuel Ben (10 secondes)
`flutter build windows` exige le **Mode développeur Windows** (symlinks des plugins) ;
l'écriture registre HKLM est bloquée pour moi. Ben : Paramètres → « Espace développeurs »
→ activer **Mode développeur**, ou en admin :
`reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" /t REG_DWORD /f /v AllowDevelopmentWithoutDevLicense /d 1`

## Étapes du build (dans l'ordre, après feu vert)

```powershell
# Environnement (à poser dans la session de build)
$env:VCPKG_ROOT = "C:\vcpkg"
$env:VCPKG_DEFAULT_HOST_TRIPLET = "x64-windows-static"
$env:LIBCLANG_PATH = "C:\Program Files\LLVM-15\bin"   # ajuster au chemin d'install
$env:Path = "C:\dev\flutter\bin;$env:USERPROFILE\.cargo\bin;$env:Path"

# 1) GROS COMPILE vcpkg (~1-3 h à froid, ~20-30 Go dans C:\vcpkg\buildtrees) — NE PAS LANCER SANS GO
cd C:\Users\Ben\Projects\doctipro-assist
C:\vcpkg\vcpkg.exe install --triplet x64-windows-static --x-install-root=C:\vcpkg\installed

# 2) Build app (~25-40 min) — NE PAS LANCER SANS GO
python .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram
# Sortie : .\flutter\build\windows\x64\runner\Release\ (rustdesk.exe + dlls)

# 3) Packaging comme le CI
Copy-Item build-extras\topmost\WindowInjection.dll .\flutter\build\windows\x64\runner\Release\
Expand-Archive build-extras\usbmmidd_v2.zip -DestinationPath .\flutter\build\windows\x64\runner\Release\
# Renommer l'exe distribué : DoctiproAssist.exe (l'installeur intégré le fait aussi tout seul)
```

## Notes / pièges
- Le moteur Flutter custom (`rustdesk/engine windows-x64-release.zip`) remplace
  `<flutter>\bin\cache\artifacts\engine\windows-x64-release\` APRÈS `flutter precache --windows`.
- Le patch dropdown s'applique dans la racine du SDK Flutter : `git apply flutter_3.24.4_dropdown_menu_enableFilter.diff`.
- `vcpkg install` en mode manifest lit `vcpkg.json` à la racine du repo (aom, libjpeg-turbo,
  opus, libvpx, libyuv, mfx-dispatch, ffmpeg[amf,nvcodec,qsv]…). ffmpeg est le plus long.
- Espace disque : ~40-60 Go au total (VS Build Tools ~7, Flutter ~3, vcpkg ~25, target cargo ~10-15).
  122 Go libres sur C: au moment du setup.
