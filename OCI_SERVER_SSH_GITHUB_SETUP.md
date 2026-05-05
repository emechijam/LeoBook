================================================================
  LEOBOOK — OCI SERVER SSH & MANAGEMENT GUIDE
  Last Updated: 2026-05-05 14:00 WAT
  Architecture: SINGLE VM — leobook-backend (84.8.218.240)
  App Release Baseline: v9.7.51+146
================================================================

================================================================
  SERVER FLEET OVERVIEW
================================================================

  VM1 — leobook-backend (SOLE PRODUCTION SERVER)
    Role   : FastAPI + PostgreSQL + Caddy HTTPS + Streamer + RL Training + Playwright
    Public : 84.8.218.240
    sslip  : https://84-8-218-240.sslip.io
    Key    : leobook-key.pem
    Path   : /home/ubuntu/leobook-backend

  NOTE: Single-VM since 2026-05-02. VM2 (84.8.222.118) permanently retired.
  All workloads — API, pipeline, training, Playwright, streamer �� run on VM1.
  No DB tunnel needed (PostgreSQL on localhost within VM1).

================================================================
  SECTION 1: KEY SETUP
================================================================

---- Windows (PowerShell) ----------------------------------------

  Copy-Item "C:\Users\Admin\Desktop\ProProjection\LeoBook\leobook-key.pem" "$env:USERPROFILE\.ssh\leobook-key.pem" -Force

---- Linux / Codespace (bash) ------------------------------------

  mkdir -p ~/.ssh
  cp /workspaces/LeoBook/leobook-key.pem ~/.ssh/leobook-key.pem
  chmod 600 ~/.ssh/leobook-key.pem

================================================================
  SECTION 2: SSH INTO VM1
================================================================

---- Windows (PowerShell) ----------------------------------------

  ssh -i "$env:USERPROFILE\.ssh\leobook-key.pem" -o StrictHostKeyChecking=no ubuntu@84.8.218.240

---- Linux / Codespace (bash) ------------------------------------

  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240

Verified: VM1 lands in /home/ubuntu/leobook-backend  (host: leobook-vnic)

================================================================
  SECTION 3: GITHUB SETUP (First-time on VM1)
================================================================

Run INSIDE VM1 (after SSH). This configures Git authentication and user identity.

---- A. Install git (if not present) ----

  sudo apt update && sudo apt install -y git

---- B. Generate ED25519 SSH key for GitHub ----

  ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519 -N ""

  Replace "your_email@example.com" with your actual GitHub email.
  The -N "" flag creates the key with NO passphrase (unattended automation).

---- C. Add SSH key to ssh-agent ----

  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519

---- D. Display public key and add to GitHub ----

  cat ~/.ssh/id_ed25519.pub

  Copy the output. Then:
    1. Go to https://github.com/settings/keys
    2. Click "New SSH key"
    3. Title: "leobook-vm1-ed25519" (or preferred label)
    4. Key type: "Authentication Key"
    5. Paste the public key (entire line)
    6. Click "Add SSH key"

---- E. Test GitHub SSH connection ----

  ssh -T git@github.com

  Expected: "Hi <username>! You've successfully authenticated..."

---- F. Configure Git user (IMPORTANT for commits) ----

  git config --global user.email "your_email@example.com"
  git config --global user.name "Your Name"

  Verify:
    git config --global user.email
    git config --global user.name

---- G. Auto-start ssh-agent on login (optional but recommended) ----

  cat >> ~/.bashrc << 'BASHEOF'
# Start ssh-agent if not running
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null
  ssh-add ~/.ssh/id_ed25519 2>/dev/null
fi
BASHEOF

  source ~/.bashrc

---- H. Clone or update LeoBook repo with SSH (one-time) ----

  If repo is HTTPS:
    cd ~/leobook-backend
    git remote set-url origin git@github.com:emechijam/LeoBook.git
    git pull origin main

  If cloning fresh:
    cd ~
    git clone git@github.com:emechijam/LeoBook.git leobook-backend
    cd leobook-backend

================================================================
  SECTION 4: VIRTUAL ENVIRONMENT
================================================================

Run INSIDE VM1 (after SSH):

  cd ~/leobook-backend
  source venv/bin/activate

One-time auto-activation (inside VM, run once):

  echo 'cd ~/leobook-backend && source venv/bin/activate' >> ~/.bashrc

================================================================
  SECTION 5: GIT PULL LATEST CODE
================================================================

---- Windows (PowerShell) ----------------------------------------

  ssh -i "$env:USERPROFILE\.ssh\leobook-key.pem" -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "cd ~/leobook-backend && git pull origin main && git log --oneline -3"

---- Linux / Codespace (bash) ------------------------------------

  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "cd ~/leobook-backend && git pull origin main && git log --oneline -3"

---- From VM1 (after GitHub setup complete) ----

  cd ~/leobook-backend && git pull origin main && git log --oneline -3

================================================================
  SECTION 6: INSTALL LEO-JOBS TOOLSET (run once after provisioning)
================================================================

  scp -i ~/.ssh/leobook-key.pem infra/leo-run infra/leo-jobs infra/leo-logs ubuntu@84.8.218.240:/tmp/
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "sudo mv /tmp/leo-run /tmp/leo-jobs /tmp/leo-logs /usr/local/bin/ && sudo sed -i 's/\r//' /usr/local/bin/leo-run /usr/local/bin/leo-jobs /usr/local/bin/leo-logs && sudo chmod +x /usr/local/bin/leo-run /usr/local/bin/leo-jobs /usr/local/bin/leo-logs && echo INSTALLED"

================================================================
  SECTION 7: PLAYWRIGHT BROWSERS (run once after provisioning)
================================================================

  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "cd ~/leobook-backend && source venv/bin/activate && playwright install chromium && playwright install-deps chromium && echo BROWSERS_OK"

NOTE: Required for: --enrich-leagues, --odds, leobook-streamer.service

================================================================
  SECTION 8: RUNNING LEOBOOK JOBS
================================================================

All jobs run on VM1 via leo-run (persistent systemd services):

  sudo leo-run --enrich-leagues --sport all --seasons 3
  sudo leo-run --odds --sport all --seasons-back 3
  sudo leo-run --train-rl --train-season all --sport all --cold
  leo-jobs                         # list all running jobs
  leo-logs                         # tail latest job
  sudo systemctl stop <unit>       # stop a specific job

  API service:
    sudo systemctl status leobook-api
    sudo systemctl restart leobook-api

  Streamer (requires Playwright — Section 7 first):
    sudo systemctl start leobook-streamer
    sudo systemctl stop leobook-streamer
    sudo systemctl status leobook-streamer

================================================================
  SECTION 9: STOP ALL JOBS (Emergency)
================================================================

  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "sudo systemctl stop leobook-api leobook-streamer 2>/dev/null; ps aux | grep -E 'Leo.py|rl_job|odds' | grep -v grep || echo Clean"

================================================================
  SECTION 10: DIAGNOSTICS
================================================================

  # API health
  curl https://84-8-218-240.sslip.io/health

  # All services
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "systemctl is-active leobook-api leobook-streamer caddy postgresql"

  # Running leo jobs
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "leo-jobs"

  # Disk / memory
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "df -h && free -h"

  # Odds job stats
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "PGPASSWORD=LeoBook_pg_20260421_rebuild_9x psql -h localhost -U leobook -d leobook -c 'SELECT status, COUNT(*) FROM odds_extraction_jobs GROUP BY status ORDER BY count DESC'"

  # Odds snapshot count
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "PGPASSWORD=LeoBook_pg_20260421_rebuild_9x psql -h localhost -U leobook -d leobook -c 'SELECT COUNT(*) AS snapshots FROM odds_snapshots'"

NOTE: Odds tables: odds_extraction_jobs, fixture_odds_catalog, odds_snapshots,
      odds_latest, odds_consensus, odds_feature_snapshots

================================================================
  SECTION 11: DAILY CHECKLIST
================================================================

  # Pull latest
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "cd ~/leobook-backend && git pull origin main && echo VM1_OK"

  # Services health
  ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "systemctl is-active leobook-api leobook-streamer caddy postgresql"

  # API health
  curl https://84-8-218-240.sslip.io/health

================================================================
  SECTION 12: SHELL ALIASES
================================================================

cat >> ~/.bashrc << 'ALIASEOF'
alias ssh-vm1='ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240'
alias vm1-pull='ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "cd ~/leobook-backend && git pull origin main"'
alias vm1-status='ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "systemctl is-active leobook-api leobook-streamer caddy postgresql"'
alias vm1-health='curl -s https://84-8-218-240.sslip.io/health | python3 -m json.tool'
alias vm1-logs='ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "leo-logs"'
alias vm1-jobs='ssh -i ~/.ssh/leobook-key.pem -o StrictHostKeyChecking=no ubuntu@84.8.218.240 "leo-jobs"'
ALIASEOF
source ~/.bashrc && echo "Aliases ready"

================================================================
  SECTION 13: TROUBLESHOOTING
================================================================

SSH ERRORS:
  "Permission denied (publickey)"   — wrong key. Use leobook-key.pem only.
  "Load key: invalid format"        — fix perms: chmod 600 ~/.ssh/leobook-key.pem

GITHUB SSH ERRORS (on VM1):
  "git@github.com: Permission denied (publickey)"
    → SSH key not added to GitHub: https://github.com/settings/keys
    → SSH key not in ssh-agent: ssh-add ~/.ssh/id_ed25519
    → Test: ssh -T git@github.com

  "Could not open a connection to your authentication agent"
    → Start ssh-agent: eval "$(ssh-agent -s)" then ssh-add ~/.ssh/id_ed25519

  "Load key: invalid format" on id_ed25519
    → Regenerate: ssh-keygen -t ed25519 -C "email@example.com" -f ~/.ssh/id_ed25519 -N ""

GIT PUSH/PULL ERRORS:
  "fatal: not a git repository"     — run: cd ~/leobook-backend
  "fatal: could not read Username"  → git config --global user.email "<email>"

PLAYWRIGHT / ENRICHMENT:
  "Executable doesn't exist at ...chromium..."
    → playwright install chromium && playwright install-deps chromium

TORCH on aarch64 (ARM64 VM):
  "No module named 'torch'"
    → DO NOT use --index-url https://download.pytorch.org/whl/cpu (x86 only)
    → Use standard PyPI: pip install torch

DB ERRORS:
  "connection refused" on 5432 → sudo systemctl start postgresql
  "password auth failed"       → check .env LEOBOOK_POSTGRES_PASSWORD

KERNEL UPGRADE WARNING:
  Safe to ignore until planned maintenance window.
  To reboot: sudo reboot  (systemd auto-restarts all services)

================================================================
  END OF GUIDE
================================================================
