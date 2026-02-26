---
Title: GitHub Actions YAML Cheat Sheet für Esp32-s3_linux
Description: Umfassende Anleitung zu GitHub Actions Workflows, YAML-Syntax, Caching, Artifacts und CI/CD-Patterns für dieses Projekt
---

# GitHub Actions YAML Cheat Sheet für Esp32-s3_linux

Dieses ausführliche Cheat Sheet bietet eine detaillierte Übersicht über GitHub Actions Workflows, speziell zugeschnitten auf das Esp32-s3_linux-Projekt. Es richtet sich an Entwickler, die CI/CD-Pipelines für Embedded-Linux-Builds verstehen und optimieren möchten.

## Was ist GitHub Actions?

GitHub Actions ist die integrierte CI/CD-Plattform von GitHub. Workflows werden als YAML-Dateien im `.github/workflows/`-Verzeichnis definiert und automatisch bei Events (Push, Pull Request, Schedule, etc.) ausgeführt.

**Vorteile für dieses Projekt:**
- Direkt in GitHub integriert (keine externe CI-Platform nötig)
- Kostenlose Minuten für öffentliche Repositories
- Docker-Container nativ unterstützt
- Matrix-Builds für verschiedene Konfigurationen (z.B. verschiedene Kernel-Versionen)
- Artifact-Storage für Build-Outputs

**Wichtige Konzepte:**
- **Workflow**: Automatisierter Prozess (definiert in `.github/workflows/*.yml`)
- **Job**: Gruppe von Steps, die auf einem Runner ausgeführt werden
- **Step**: Einzelner Task (Befehl oder Action)
- **Runner**: Virtuelle Maschine, die Jobs ausführt (GitHub-hosted oder self-hosted)
- **Action**: Wiederverwendbare Unit (z.B. `actions/checkout@v4`)

## Grundlegende Workflow-Struktur

### Minimales Workflow-Beispiel
```yaml
name: CI

# Wann wird der Workflow ausgeführt?
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

# Jobs die ausgeführt werden
jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Run build
        run: |
          python --version
          echo "Building..."
```

### Detaillierte Erklärung der Struktur

#### name - Workflow-Name
```yaml
name: Buildroot CI/CD
```
**Erklärung:** Angezeigter Name in der GitHub Actions UI. Sollte beschreibend sein.

#### on - Trigger-Events

**Push-Event:**
```yaml
on:
  push:
    branches:
      - main
      - 'release/*'  # Wildcards möglich
    paths:
      - 'src/**'     # Nur wenn Dateien in src/ geändert
      - '!docs/**'   # Aber nicht bei Docs
    tags:
      - 'v*'         # Bei Version-Tags
```

**Pull Request:**
```yaml
on:
  pull_request:
    branches: [ main ]
    types:
      - opened       # Neue PR
      - synchronize  # Neue Commits in PR
      - reopened     # PR wurde wiedereröffnet
```

**Zeitplan (Cron):**
```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # Täglich um 2:00 UTC
    # ┌───────────── Minute (0 - 59)
    # │ ┌───────────── Stunde (0 - 23)
    # │ │ ┌───────────── Tag des Monats (1 - 31)
    # │ │ │ ┌───────────── Monat (1 - 12)
    # │ │ │ │ ┌───────────── Wochentag (0 - 6, Sonntag = 0)
    # │ │ │ │ │
    # * * * * *
```

**Beispiel: Nächtlicher Build**
```yaml
on:
  schedule:
    - cron: '0 3 * * 1'  # Jeden Montag um 3:00 UTC

  # Manuelle Auslösung erlauben
  workflow_dispatch:
    inputs:
      buildroot_version:
        description: 'Buildroot version to build'
        required: true
        default: '2024.02'
```

**Mehrere Events kombinieren:**
```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # Manuell triggern
```

#### jobs - Workflow-Jobs

**Einfacher Job:**
```yaml
jobs:
  build:
    name: Build Buildroot
    runs-on: ubuntu-22.04  # Spezifische Ubuntu-Version
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```

**Job mit Timeout:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 180  # 3 Stunden max
    
    steps:
      - name: Long build
        run: make -j$(nproc)
        timeout-minutes: 120  # Step-spezifisches Timeout
```

**Warum Timeouts wichtig sind:**
- Verhindert hängende Jobs (verschwendete CI-Minuten)
- Standard-Timeout: 360 Minuten (6 Stunden)
- Für große Builds: Timeout erhöhen

**Job-Abhängigkeiten:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."
  
  test:
    needs: build  # Wartet auf "build" Job
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."
  
  deploy:
    needs: [build, test]  # Wartet auf beide
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

**Visualisierung:**
```
build → test ↘
               → deploy
```

#### steps - Job-Schritte

**Run-Command:**
```yaml
steps:
  - name: Single line command
    run: echo "Hello World"
  
  - name: Multi-line command
    run: |
      echo "Line 1"
      echo "Line 2"
      ls -la
```

**Mit Working Directory:**
```yaml
steps:
  - name: Build in subdirectory
    run: make -j$(nproc)
    working-directory: ./buildroot
```

**Mit Umgebungsvariablen:**
```yaml
steps:
  - name: Build with env vars
    run: make -j${MAKE_JOBS}
    env:
      MAKE_JOBS: 4
      CROSS_COMPILE: xtensa-linux-
```

**Conditional Steps (Bedingungen):**
```yaml
steps:
  - name: Only on main branch
    if: github.ref == 'refs/heads/main'
    run: echo "Main branch!"
  
  - name: Only on PR
    if: github.event_name == 'pull_request'
    run: echo "Pull Request!"
  
  - name: Only on success
    if: success()
    run: echo "Previous steps succeeded"
  
  - name: Always run (even on failure)
    if: always()
    run: echo "Cleanup..."
```

**Wichtige Bedingungsfunktionen:**
- `success()` - Alle vorherigen Steps erfolgreich
- `failure()` - Mindestens ein Step fehlgeschlagen
- `always()` - Immer ausführen
- `cancelled()` - Workflow wurde abgebrochen

## Häufig verwendete Actions

### actions/checkout@v4 - Repository auschecken

```yaml
steps:
  - name: Checkout repository
    uses: actions/checkout@v4
```

**Standard-Verhalten:**
- Checkt aktuellen Branch/PR aus
- Shallow clone (nur letzter Commit)
- Keine Submodules

**Erweiterte Optionen:**
```yaml
- uses: actions/checkout@v4
  with:
    # Komplette History (alle Commits)
    fetch-depth: 0
    
    # Submodules mit auschecken
    submodules: 'recursive'
    
    # Spezifischer Branch/Ref
    ref: 'develop'
    
    # In Unterverzeichnis auschecken
    path: 'source'
    
    # LFS-Dateien pullen
    lfs: true
```

**Für dieses Projekt (mit Submodules):**
```yaml
- uses: actions/checkout@v4
  with:
    submodules: 'recursive'
    fetch-depth: 0  # Für git describe
```

### actions/cache@v4 - Build-Cache

**Warum Caching?**
- Beschleunigt Builds erheblich (apt-get, pip, npm, etc.)
- Spart CI-Minuten
- Reduziert externe Downloads

**Beispiel: APT-Cache**
```yaml
- name: Cache APT packages
  uses: actions/cache@v4
  with:
    path: /var/cache/apt/archives
    key: ${{ runner.os }}-apt-${{ hashFiles('**/Dockerfile') }}
    restore-keys: |
      ${{ runner.os }}-apt-
```

**Erklärung:**
- `path`: Was wird gecached
- `key`: Cache-Schlüssel (bei Änderung wird Cache invalidiert)
- `restore-keys`: Fallback-Schlüssel wenn exakter Match fehlt

**Beispiel: pip-Cache**
```yaml
- name: Cache pip packages
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

**Beispiel: Buildroot DL-Cache (Downloads)**
```yaml
- name: Cache Buildroot downloads
  uses: actions/cache@v4
  with:
    path: buildroot/dl
    key: buildroot-dl-${{ hashFiles('**/buildroot.esp32s3.defconfig') }}
    restore-keys: |
      buildroot-dl-
```

**Wichtig:**
- Cache ist max. 10GB pro Repository
- Nicht abgerufene Caches werden nach 7 Tagen gelöscht
- Cache ist zwischen Branches geteilt (main kann Cache von feature-branch nutzen)

**Cache-Hit prüfen:**
```yaml
- name: Cache dependencies
  id: cache-deps
  uses: actions/cache@v4
  with:
    path: ./deps
    key: deps-${{ hashFiles('deps.lock') }}

- name: Install dependencies
  if: steps.cache-deps.outputs.cache-hit != 'true'
  run: ./install-deps.sh
```

### actions/upload-artifact@v4 - Artifacts hochladen

```yaml
- name: Upload build artifacts
  uses: actions/upload-artifact@v4
  with:
    name: buildroot-images
    path: |
      buildroot/output/images/
      buildroot/output/build.log
    retention-days: 30  # Wie lange behalten?
    if-no-files-found: error  # Fehler wenn nichts gefunden
```

**Optionen für `if-no-files-found`:**
- `error` - Job schlägt fehl (Standard für wichtige Artifacts)
- `warn` - Warnung aber kein Fehler
- `ignore` - Stillschweigend ignorieren

**Mehrere Artifacts:**
```yaml
- name: Upload kernel image
  uses: actions/upload-artifact@v4
  with:
    name: kernel-image
    path: buildroot/output/images/Image

- name: Upload rootfs
  uses: actions/upload-artifact@v4
  with:
    name: rootfs
    path: buildroot/output/images/rootfs.tar
```

### actions/download-artifact@v4 - Artifacts herunterladen

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make
      - uses: actions/upload-artifact@v4
        with:
          name: binary
          path: ./binary
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: binary
          path: ./downloaded
      
      - run: ./downloaded/binary --test
```

**Alle Artifacts eines Workflows herunterladen:**
```yaml
- uses: actions/download-artifact@v4
  # Ohne name → lädt alle Artifacts
```

### actions/setup-python@v5 - Python einrichten

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'  # Automatisches pip-Caching
```

**Matrix mit mehreren Python-Versionen:**
```yaml
strategy:
  matrix:
    python-version: ['3.9', '3.10', '3.11', '3.12']

steps:
  - uses: actions/setup-python@v5
    with:
      python-version: ${{ matrix.python-version }}
```

### docker/build-push-action@v5 - Docker Images bauen

```yaml
- name: Build Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    push: false  # Nur bauen, nicht pushen
    tags: esp32-buildroot:latest
    cache-from: type=gha  # GitHub Actions Cache
    cache-to: type=gha,mode=max
```

**Mit Push zu Registry:**
```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      myuser/esp32-buildroot:latest
      myuser/esp32-buildroot:${{ github.sha }}
```

## Matrix-Builds - Parallele Builds mit Variationen

Matrix-Builds ermöglichen das Testen verschiedener Konfigurationen parallel.

### Einfache Matrix
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        gcc: [11, 12, 13]
        kernel: ["5.15", "6.1", "6.6"]
    
    steps:
      - name: Build with GCC ${{ matrix.gcc }} and Kernel ${{ matrix.kernel }}
        run: |
          echo "GCC: ${{ matrix.gcc }}"
          echo "Kernel: ${{ matrix.kernel }}"
          make GCC_VERSION=${{ matrix.gcc }} KERNEL_VERSION=${{ matrix.kernel }}
```

**Ergebnis:** 9 parallele Jobs (3 GCC × 3 Kernel Versionen)

### Matrix mit include/exclude
```yaml
strategy:
  matrix:
    os: [ubuntu-22.04, ubuntu-20.04]
    compiler: [gcc, clang]
    
    # Spezielle Konfiguration hinzufügen
    include:
      - os: ubuntu-22.04
        compiler: gcc
        gcc-version: 12
    
    # Bestimmte Kombination ausschließen
    exclude:
      - os: ubuntu-20.04
        compiler: clang
```

### Matrix mit fail-fast
```yaml
strategy:
  fail-fast: false  # Andere Jobs weiterlaufen lassen bei Fehler
  matrix:
    config:
      - { name: "Quick", args: "QUICK_CHECK=true" }
      - { name: "Full", args: "" }

steps:
  - name: Build ${{ matrix.config.name }}
    run: make ${{ matrix.config.args }}
```

**Standard `fail-fast: true`:** Alle Jobs werden abgebrochen wenn einer fehlschlägt.
**Mit `fail-fast: false`:** Alle Jobs laufen durch, auch wenn einer fehlschlägt (nützlich um alle Fehler zu sehen).

### Matrix aus JSON
```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          # Dynamische Matrix generieren
          echo "matrix={\"config\":[\"debug\",\"release\"]}" >> $GITHUB_OUTPUT
  
  build:
    needs: setup
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}
    steps:
      - run: echo "Building ${{ matrix.config }}"
```

## Umgebungsvariablen und Secrets

### Umgebungsvariablen setzen

**Workflow-Level:**
```yaml
env:
  BUILDROOT_VERSION: 2024.02
  MAKE_JOBS: 4

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Buildroot: $BUILDROOT_VERSION"
```

**Job-Level:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      CC: gcc
      ARCH: xtensa
    steps:
      - run: echo "Compiler: $CC, Arch: $ARCH"
```

**Step-Level:**
```yaml
steps:
  - name: Build
    env:
      CROSS_COMPILE: xtensa-linux-
    run: make
```

### GitHub-Kontext-Variablen

```yaml
steps:
  - name: Print context
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Ref: ${{ github.ref }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
```

**Nützliche Variablen:**
- `github.repository` → `OehlF/Esp32-s3_linux`
- `github.ref` → `refs/heads/main`
- `github.sha` → Commit-SHA (z.B. `abc123...`)
- `github.event_name` → `push`, `pull_request`, etc.
- `runner.os` → `Linux`, `Windows`, `macOS`
- `runner.arch` → `X64`, `ARM64`

### Secrets verwenden

**In Repository Settings → Secrets and variables → Actions:**
- Secrets hinzufügen (z.B. `DOCKERHUB_TOKEN`)

**In Workflow verwenden:**
```yaml
steps:
  - name: Use secret
    env:
      TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
    run: |
      # Secret ist verfügbar als Umgebungsvariable
      echo "Token length: ${#TOKEN}"
      # Niemals secrets direkt ausgeben!
      # echo "$TOKEN"  ❌ NIEMALS!
```

**Wichtig:**
- Secrets werden in Logs automatisch maskiert (`***`)
- Verwenden Sie Secrets nur in env oder with, nicht direkt in run
- Für selbst-gehostete Runner: Secrets nicht auf unsicheren Systemen verwenden

**Sichere Nutzung:**
```yaml
# ✅ RICHTIG
- name: Login
  env:
    PASSWORD: ${{ secrets.PASSWORD }}
  run: echo "$PASSWORD" | docker login -u user --password-stdin

# ❌ FALSCH
- name: Login
  run: docker login -u user -p ${{ secrets.PASSWORD }}
  # Secret könnte in Process-Liste sichtbar sein!
```

## Runner-Auswahl und Self-Hosted Runner

### GitHub-Hosted Runner

**Verfügbare Images:**
```yaml
runs-on: ubuntu-latest      # Ubuntu (neueste LTS)
runs-on: ubuntu-22.04       # Ubuntu 22.04 (spezifisch)
runs-on: ubuntu-20.04       # Ubuntu 20.04
runs-on: windows-latest     # Windows Server
runs-on: macos-latest       # macOS (ARM)
runs-on: macos-13           # macOS (Intel)
```

**Specs (Stand 2024):**
- **CPU:** 2-4 Cores
- **RAM:** 7-14 GB
- **Disk:** 14-90 GB SSD
- **Limit:** 6 Stunden pro Job

**Für große Buildroot-Builds:**
```yaml
runs-on: ubuntu-latest
# Eventuell Self-Hosted Runner nutzen mit mehr RAM/CPU
```

### Self-Hosted Runner

**Einrichtung:**
1. GitHub Repository → Settings → Actions → Runners
2. "New self-hosted runner" klicken
3. Anweisungen folgen (Download + Setup-Script)

**In Workflow verwenden:**
```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    
    steps:
      - uses: actions/checkout@v4
      - run: make -j$(nproc)
```

**Labels kombinieren:**
```yaml
runs-on: [self-hosted, linux, x64, gpu]
# Benötigt Runner mit allen Labels
```

**Vorteile Self-Hosted:**
- Mehr RAM/CPU/Disk verfügbar
- Persistent caching möglich
- Zugriff auf lokale Hardware (z.B. ESP32-Board)
- Unbegrenzte Minuten

**Nachteile:**
- Wartung erforderlich
- Sicherheitsrisiko (externe PR-Code läuft auf Ihrer Maschine!)
- Keine automatische Isolation

**Sicherheit für Self-Hosted Runner:**
```yaml
jobs:
  build:
    # Nur für eigene PRs, nicht für Forks
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: [self-hosted, linux]
```

## Wiederverwendbare Workflows

### Workflow aufrufen

**.github/workflows/build.yml:**
```yaml
name: Build Workflow

on:
  workflow_call:
    inputs:
      buildroot-version:
        required: true
        type: string
    outputs:
      image-path:
        value: ${{ jobs.build.outputs.image }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.build.outputs.path }}
    
    steps:
      - uses: actions/checkout@v4
      
      - id: build
        run: |
          make BUILDROOT_VERSION=${{ inputs.buildroot-version }}
          echo "path=output/images/Image" >> $GITHUB_OUTPUT
```

**.github/workflows/ci.yml:**
```yaml
name: CI

on: [push, pull_request]

jobs:
  call-build:
    uses: ./.github/workflows/build.yml
    with:
      buildroot-version: '2024.02'
  
  test:
    needs: call-build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Image: ${{ needs.call-build.outputs.image-path }}"
```

**Vorteile:**
- DRY (Don't Repeat Yourself)
- Zentrale Wartung
- Konsistenz über Workflows hinweg

## Berechtigungen (Permissions)

**Least-Privilege-Prinzip:**
```yaml
permissions:
  contents: read  # Repository lesen
  packages: write # GitHub Packages schreiben
  pull-requests: write  # PR kommentieren

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

**Verfügbare Berechtigungen:**
- `actions` - GitHub Actions Artifacts/Workflows
- `checks` - Checks API
- `contents` - Repository-Inhalte
- `deployments` - Deployments
- `issues` - Issues
- `packages` - GitHub Packages
- `pull-requests` - Pull Requests
- `security-events` - Security Events (CodeQL)

**Alle Berechtigungen entziehen:**
```yaml
permissions: {}
# Minimal permissions - verhindert versehentliche Schreibzugriffe
```

## Häufige Fehler und Problemlösungen

### Problem: "Resource not accessible by integration"

**Symptom:**
```
Error: Resource not accessible by integration
```

**Ursache:** Fehlende Berechtigungen für Token.

**Lösung:**
```yaml
permissions:
  contents: write  # Oder was auch immer benötigt wird
```

### Problem: Job schlägt fehl mit "Disk quota exceeded"

**Symptom:**
```
Error: No space left on device
```

**Ursache:** Runner hat zu wenig Disk-Space (GitHub-hosted: ~14GB nutzbar).

**Lösungen:**

**1. Cleanup vor Build:**
```yaml
- name: Free disk space
  run: |
    df -h
    sudo rm -rf /usr/share/dotnet
    sudo rm -rf /usr/local/lib/android
    sudo rm -rf /opt/ghc
    sudo rm -rf /opt/hostedtoolcache/CodeQL
    df -h
```

**2. Docker Cleanup:**
```yaml
- name: Docker cleanup
  run: docker system prune -af
```

**3. Self-Hosted Runner nutzen:**
```yaml
runs-on: [self-hosted, linux, large-disk]
```

### Problem: Cache wird nicht genutzt

**Symptom:** Jeder Build lädt alles neu herunter.

**Ursachen & Lösungen:**

**1. Cache-Key ändert sich:**
```yaml
# ❌ SCHLECHT - Key ändert sich bei jedem Commit
key: ${{ runner.os }}-cache-${{ github.sha }}

# ✅ GUT - Key basiert auf Dependency-Datei
key: ${{ runner.os }}-cache-${{ hashFiles('**/package-lock.json') }}
```

**2. Cache existiert nicht mehr (7 Tage Limit):**
```yaml
restore-keys: |
  ${{ runner.os }}-cache-
  # Nutzt ältere Caches als Fallback
```

**3. Cache-Größe überschritten (10GB Limit):**
- Selektiver cachen (nur wichtige Directories)
- Alte Branches löschen (deren Caches werden auch gelöscht)

### Problem: Matrix-Build zu viele Jobs

**Symptom:** 20+ Jobs laufen parallel, CI-Minuten verbraucht.

**Lösung: max-parallel limitieren:**
```yaml
strategy:
  max-parallel: 4  # Maximal 4 gleichzeitig
  matrix:
    version: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

### Problem: Secrets in Logs sichtbar

**Symptom:** Secret-Wert erscheint in Logs (base64-encoded o.ä.).

**Ursache:** Secret wurde transformiert (base64, etc.) und ist nicht mehr maskiert.

**Lösung:**
```yaml
- name: Mask transformed secrets
  run: |
    SECRET_BASE64=$(echo -n "${{ secrets.PASSWORD }}" | base64)
    echo "::add-mask::$SECRET_BASE64"
    # Jetzt ist auch base64-Version maskiert
    echo "Encoded: $SECRET_BASE64"  # → Encoded: ***
```

### Problem: "unable to access repository"

**Symptom:**
```
Error: unable to access 'https://github.com/user/repo/': Could not resolve host
```

**Ursachen:**
1. Repository ist privat und Token fehlt
2. Netzwerkprobleme (transient)
3. Token hat abgelaufen

**Lösungen:**
```yaml
# Private Repositories
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.GITHUB_TOKEN }}  # Auto-provided
    # Oder für andere Repos:
    token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

### Problem: Build erfolgreich lokal, aber nicht in CI

**Häufige Ursachen:**

**1. Unterschiedliche Umgebungen:**
```yaml
# In CI environment debuggen
- name: Debug environment
  run: |
    uname -a
    cat /etc/os-release
    env | sort
    which gcc
    gcc --version
```

**2. Fehlende Dependencies:**
```yaml
# Explizit installieren
- name: Install dependencies
  run: |
    sudo apt-get update
    sudo apt-get install -y build-essential libncurses-dev
```

**3. Timing/Race Conditions:**
```bash
# Mehr parallele Jobs lokal als in CI
make -j$(nproc)  # Lokal: 16 Cores, CI: 2 Cores
# → Race Condition nur lokal sichtbar oder umgekehrt
```

### Problem: Timeout in langen Builds

**Symptom:**
```
Error: The job running on runner has exceeded the maximum execution time of 360 minutes.
```

**Lösungen:**

**1. Timeout erhöhen:**
```yaml
jobs:
  build:
    timeout-minutes: 480  # 8 Stunden
```

**2. Build optimieren:**
```yaml
- name: Build with cache
  uses: actions/cache@v4
  with:
    path: buildroot/dl
    key: buildroot-dl-${{ hashFiles('config/*') }}

- run: make -j$(nproc)  # Paralleler Build
```

**3. Matrix aufteilen:**
```yaml
# Statt ein großer Build, mehrere kleine
strategy:
  matrix:
    component: [kernel, rootfs, bootloader]
```

## Debugging-Strategien

### 1. Aktivieren Sie Debug-Logging

**In Repository Settings:**
- Settings → Secrets and variables → Actions
- New repository secret: `ACTIONS_STEP_DEBUG` = `true`
- Und/oder: `ACTIONS_RUNNER_DEBUG` = `true`

**Ergebnis:** Sehr detaillierte Logs für jeden Step.

### 2. SSH-Zugang zu Runner (tmate)

```yaml
- name: Setup tmate session
  if: failure()  # Nur bei Fehler
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 30
```

**Ergebnis:** SSH-Zugang zum Runner nach Fehler (Logs zeigen SSH-Befehl).

**Wichtig:** Nur in privaten Repos oder mit Authentifizierung nutzen!

### 3. Artifacts für Debugging

```yaml
- name: Upload logs on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: debug-logs
    path: |
      buildroot/output/build.log
      /var/log/
      ~/.npm/_logs/
```

### 4. Environment und Disk-Space prüfen

```yaml
- name: Debug environment
  run: |
    echo "::group::System Info"
    uname -a
    cat /etc/os-release
    echo "::endgroup::"
    
    echo "::group::Disk Space"
    df -h
    echo "::endgroup::"
    
    echo "::group::Memory"
    free -h
    echo "::endgroup::"
    
    echo "::group::Environment"
    env | sort
    echo "::endgroup::"
```

**`::group::` und `::endgroup::`:** Erstellt einklappbare Sektionen in Logs.

### 5. Matrix-Builds einzeln testen

```yaml
strategy:
  matrix:
    config: [debug, release, asan]

steps:
  - name: Build ${{ matrix.config }}
    run: make CONFIG=${{ matrix.config }}
  
  - name: Upload on failure
    if: failure()
    uses: actions/upload-artifact@v4
    with:
      name: failed-${{ matrix.config }}
      path: build/
```

## Best Practices

### 1. Nutzen Sie Caching aggressiv

```yaml
# Buildroot downloads
- uses: actions/cache@v4
  with:
    path: buildroot/dl
    key: br-dl-${{ hashFiles('config/*.defconfig') }}

# ccache (Compiler Cache)
- uses: actions/cache@v4
  with:
    path: ~/.ccache
    key: ccache-${{ github.sha }}
    restore-keys: ccache-
```

### 2. Fail-Fast für schnelles Feedback

```yaml
strategy:
  fail-fast: true  # Stoppe bei erstem Fehler
  matrix:
    test: [unit, integration, e2e]
```

**Außer:** Sie wollen alle Fehler sehen → `fail-fast: false`

### 3. Conditional Jobs für optimale Ressourcennutzung

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      docs: ${{ steps.filter.outputs.docs }}
      code: ${{ steps.filter.outputs.code }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            docs:
              - 'docs/**'
            code:
              - 'src/**'
  
  build-docs:
    needs: changes
    if: needs.changes.outputs.docs == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: build-docs
  
  build-code:
    needs: changes
    if: needs.changes.outputs.code == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: make
```

### 4. Dependabot für Action-Updates

**.github/dependabot.yml:**
```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

**Resultat:** Automatische PRs wenn Actions Updates haben (z.B. `actions/checkout@v4` → `@v5`)

### 5. Wiederverwendbare Workflows

**Statt:**
- Gleicher Code in 5 Workflows

**Besser:**
- Ein wiederverwendbarer Workflow
- 5 Workflows rufen ihn auf

### 6. Comprehensive Workflow-Template

```yaml
name: Buildroot CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * 1'  # Wöchentlich
  workflow_dispatch:

env:
  BUILDROOT_VERSION: 2024.02

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 240
    
    strategy:
      fail-fast: false
      matrix:
        config:
          - { name: "Debug", args: "BR2_DEBUG=y" }
          - { name: "Release", args: "" }
    
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      
      - name: Free disk space
        run: |
          sudo rm -rf /usr/share/dotnet
          sudo rm -rf /usr/local/lib/android
          df -h
      
      - name: Cache Buildroot downloads
        uses: actions/cache@v4
        with:
          path: buildroot/dl
          key: br-dl-${{ hashFiles('config/*.defconfig') }}
      
      - name: Build ${{ matrix.config.name }}
        env:
          MAKE_ARGS: ${{ matrix.config.args }}
        run: |
          make BR2_DEFCONFIG=config/buildroot.esp32s3.defconfig defconfig
          make -j$(nproc) $MAKE_ARGS
      
      - name: Upload artifacts
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: images-${{ matrix.config.name }}
          path: buildroot/output/images/
          retention-days: 30
      
      - name: Upload logs on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: logs-${{ matrix.config.name }}
          path: buildroot/output/build.log
```

## Weiterführende Ressourcen

### Offizielle Dokumentation
- **GitHub Actions Dokumentation:** https://docs.github.com/en/actions
  - Vollständige Referenz aller Features
- **Workflow Syntax:** https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
  - Komplette YAML-Syntax-Referenz
- **Context and Expression Syntax:** https://docs.github.com/en/actions/learn-github-actions/contexts
  - Alle verfügbaren Variablen und Funktionen

### Actions Marketplace
- **GitHub Marketplace:** https://github.com/marketplace?type=actions
  - Tausende vorgefertigte Actions
- **Awesome Actions:** https://github.com/sdras/awesome-actions
  - Kuratierte Liste nützlicher Actions

### Best Practices & Tutorials
- **GitHub Actions Best Practices:** https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions
  - Offizielle Security-Richtlinien
- **GitHub Skills:** https://skills.github.com/
  - Interaktive Tutorials
- **GitHub Actions Cookbook:** https://github.com/Authoring-Tool-Ensemble/actions-cookbook
  - Rezept-Sammlung für häufige Szenarien

### Monitoring und Debugging
- **act:** https://github.com/nektos/act
  - Führt GitHub Actions lokal mit Docker aus
  - Ideal zum Testen vor Push
- **GitHub Actions Toolkit:** https://github.com/actions/toolkit
  - SDKs zum Schreiben eigener Actions

### Security
- **Scorecard:** https://github.com/ossf/scorecard
  - Automatische Security-Checks für Actions
- **GitHub Actions Security Guide:** https://securitylab.github.com/research/github-actions-preventing-pwn-requests/
  - Verhindert PWN-Requests und ähnliche Angriffe

### Spezifisch für dieses Projekt
- **Buildroot CI Examples:** https://gitlab.com/buildroot.org/buildroot/-/blob/master/.gitlab-ci.yml
  - CI/CD für Buildroot (GitLab, aber übertragbar)
- **Docker Build Optimization:** https://docs.docker.com/build/ci/github-actions/
  - Docker-spezifische GitHub Actions Optimierungen

### Community
- **GitHub Community Forum:** https://github.com/orgs/community/discussions
  - Fragen zu GitHub Actions
- **Stack Overflow:** Tag `github-actions`
- **Awesome CI/CD:** https://github.com/cicdops/awesome-ciandcd
  - Allgemeine CI/CD-Ressourcen

### Tools zur Workflow-Visualisierung
- **GitHub Actions VSCode Extension:** https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-github-actions
  - Syntax-Highlighting, Autocomplete, Workflow-Graph
- **Workflow Diagram Generator:** https://githubactions.dev/
  - Visualisiert Workflow als Diagramm
