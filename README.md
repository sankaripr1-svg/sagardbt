# dbt_demo: RAW → STAGING → CORE with DQ quarantine (Snowflake + GitHub)

A small, self-contained dbt Core project for trying the full workflow end to end on a
personal laptop with a Snowflake trial account. All data is synthetic; no real patient data.

## What it builds

```
seeds (RAW, all text)          staging                              core / audit
───────────────────            ──────────────────────────────       ────────────────────────
raw_patients (1,057)  ─┐       stg_patient_validated (view)    ┌──► patient_hist (snapshot, SCD2)
raw_patients_batch2   ─┴──►      every raw row once + dq_reason├──► stg_patient (clean, deduped)
  (25, optional)                                               ├──► dq_quarantine (failed a rule)
                                                               └──► dq_duplicates (older copies)
                                                                    audit_row_reconciliation (view)
```

Every RAW row ends up in exactly one of `stg_patient`, `dq_quarantine` or `dq_duplicates`,
so the counts always add up. The test `tests/assert_patient_rows_reconcile.sql` fails the
build if they don't.

| Run | RAW | STAGING | QUARANTINE | DUPLICATES | Difference | PATIENT_HIST |
|---|---|---|---|---|---|---|
| Batch 1 | 1,057 | 970 | 42 | 45 | 0 | 970 |
| Batch 1 + 2 | 1,082 | 975 | 42 | 65 | 0 | 995 (975 current + 20 history) |

DQ rules (first failing rule wins, one quarantine row per raw row): missing patient_id,
unparseable DOB, future DOB, gender not M/F/U, invalid email.

## Practice workbook (encounters, providers)

New in this version: healthcare practice data and 14 scenario exercises.

| Seed | Rows | Notes |
|---|---|---|
| raw/raw_encounters | 4,000 | Day-1 landing file, all text, deliberately messy |
| raw/raw_encounters_day2 | 230 | 150 new + 80 changed encounters (`--vars "{load_day: 2}"`) |
| raw/raw_providers | 40 | Two invalid NPIs |
| reference/ref_icd10_chapters | 25 | ICD-10 first letter to chapter |
| reference/ref_encounter_types | 3 | IP / OP / ER |

The exercises live in the workbook doc. Reference answers are in `solutions/`
(see `solutions/README.md`); dbt ignores that folder until you copy files out.

Warnings such as "Configuration paths exist ... which do not apply to any resources"
are expected until you create the `intermediate/`, `marts/` and `staging/encounters/` models.

## Folder layout

```
dbt_demo/
├─ dbt_project.yml
├─ requirements.txt            pinned dbt-core + dbt-snowflake
├─ profiles.example.yml        copy to C:\Users\<you>\.dbt\profiles.yml
├─ seeds/                      synthetic patient CSVs
├─ models/staging/             validation, quarantine, dedup, clean staging
├─ models/audit/               row-count reconciliation view
├─ snapshots/patient_hist.yml  SCD Type 2 history
├─ tests/                      reconciliation test
├─ macros/drop_ci_schemas.sql  CI cleanup
├─ ci/profiles.yml             CI connection (env vars only)
├─ .github/workflows/dbt_ci.yml
└─ setup/                      Snowflake setup + result-check SQL
```

## Steps

Software installation (Python, Git, VS Code, extensions) is covered in Part 1 of the setup guide.

### 1. Snowflake trial
1. Sign up at signup.snowflake.com (Standard or Enterprise, any cloud, a nearby region).
2. Generate a key pair in Git Bash:
   ```bash
   mkdir -p ~/.ssh/snowflake && cd ~/.ssh/snowflake
   openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 des3 -inform PEM -out rsa_key.p8
   openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
   ```
   Repeat in another folder (e.g. `~/.ssh/snowflake_ci`) for the CI user.
3. In Snowsight, open a worksheet, paste `setup/01_snowflake_setup.sql`, replace the
   `<...>` placeholders, and run it.

### 2. Project and Python environment (PowerShell)
```powershell
# unzip to C:\dev\dbt_demo, then:
cd C:\dev\dbt_demo
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
dbt --version
```

### 3. Connection profile
```powershell
mkdir $env:USERPROFILE\.dbt -Force
copy profiles.example.yml $env:USERPROFILE\.dbt\profiles.yml
notepad $env:USERPROFILE\.dbt\profiles.yml        # fill account, user, key path
[Environment]::SetEnvironmentVariable("SNOWFLAKE_KEY_PASSPHRASE", "your-passphrase", "User")
```
Close and reopen the terminal (and VS Code) so the variable is picked up, then:
```powershell
dbt debug          # expect: All checks passed!
```

### 4. Run end to end
```powershell
dbt build                                   # batch 1: seed → models → snapshot → tests
dbt build --vars "{load_batch2: true}"      # batch 2: 20 changes + 5 new patients
dbt docs generate
dbt docs serve                              # lineage graph in the browser
```
Then run `setup/02_check_results.sql` in Snowsight to see each layer, the quarantine
reasons, and patients with history versions.

Objects land in `DEMO_DB.DBT_DEV_RAW`, `DBT_DEV_STAGING`, `DBT_DEV_CORE` and `DBT_DEV_AUDIT`
(dbt prefixes custom schemas with your profile schema so each developer gets their own copy).

Try breaking it: in `models/staging/dq_duplicates.sql` change `> 1` to `> 2` so some duplicates
are dropped silently, rerun `dbt build`, and watch `assert_patient_rows_reconcile` fail.

### 5. GitHub (personal, private repo)
1. Create an empty private repo `dbt_demo` on GitHub (no README, no .gitignore).
2. Push:
   ```powershell
   git init
   git add .
   git status                  # confirm no profiles.yml, .p8 or .venv is listed
   git commit -m "Initial dbt demo project"
   git branch -M main
   git remote add origin https://github.com/<your-username>/dbt_demo.git
   git push -u origin main
   ```
3. Add repository secrets (Settings → Secrets and variables → Actions):

   | Secret | Value |
   |---|---|
   | `SNOWFLAKE_ACCOUNT` | `myorg-myaccount` |
   | `SNOWFLAKE_USER` | `DBT_CI_USER` |
   | `SNOWFLAKE_ROLE` | `DBT_DEMO_ROLE` |
   | `SNOWFLAKE_WAREHOUSE` | `DBT_DEMO_WH` |
   | `SNOWFLAKE_PRIVATE_KEY` | full contents of the CI `rsa_key.p8`, including BEGIN/END lines |
   | `SNOWFLAKE_KEY_PASSPHRASE` | passphrase of the CI key |

### 6. Test the pull-request flow
```powershell
git checkout -b feature/add-rule
# make a change, e.g. add a new DQ rule in models/staging/stg_patient_validated.sql
git add . ; git commit -m "Add DQ rule" ; git push -u origin feature/add-rule
```
Open a pull request on GitHub. The **dbt CI** workflow builds batch 1 and batch 2 in schemas
named `DBT_CI_PR_<number>_*`, runs all tests, then drops those schemas. You can also run it
manually from the Actions tab (workflow_dispatch).

Optional: Settings → Branches → add a rule for `main` requiring the `dbt-build` check.

## Troubleshooting

| Problem | Fix |
|---|---|
| `dbt` not recognized | Activate the environment: `.\.venv\Scripts\Activate.ps1` |
| Scripts disabled in PowerShell | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `env_var` SNOWFLAKE_KEY_PASSPHRASE not set | Reopen the terminal/VS Code after setting it |
| JWT token invalid | Public key not registered or wrong user; check `RSA_PUBLIC_KEY_FP` in `DESC USER` |
| CI fails at private key | Secret must contain the whole `.p8` file including BEGIN/END lines |
| Snapshot shows no history | Run batch 1 first, then batch 2; the snapshot compares against the previous run |
