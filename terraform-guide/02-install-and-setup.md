# 02 — Install and set up

## What you need

Four things. Only the first is Terraform itself.

| Tool | Why | Version in `arifa-infra` |
|------|-----|--------------------------|
| Terraform | The thing itself | **1.17.1** (`.terraform-version`) |
| AWS CLI v2 | Credentials and lookups | v2, unpinned |
| tflint | Catches mistakes `validate` cannot | **0.65.0** |
| trivy | Misconfigurations and committed secrets | **0.76.1** |

The versions matter. `arifa-infra/envs/dev/versions.tf` says:

```hcl
required_version = "~> 1.17.0"
```

`~> 1.17.0` means **`>= 1.17.0` and `< 1.18.0`**. Terraform 1.16.6 is refused
outright:

```
Error: Unsupported Terraform Core version
This configuration does not support Terraform version 1.16.6.
```

That is deliberate. Everyone on the team runs the same minor version, so plans
are identical everywhere.

## Install on Windows

You have Git Bash already (`C:\Program Files\Git\bin\bash.exe`). Use it for
everything in this guide — the scripts are bash.

**With a package manager** (easiest, if you have one):

```powershell
# scoop
scoop install terraform tflint trivy
scoop install aws

# or chocolatey, from an elevated prompt
choco install terraform tflint trivy awscli
```

Package managers may not have the exact pinned version. Check with
`terraform -version` and fall back to the manual route if it is wrong.

**Manually**, which guarantees the right version. Download, unzip, and put the
binary on your `PATH`:

| Tool | Release page |
|------|--------------|
| Terraform 1.17.1 | `https://releases.hashicorp.com/terraform/1.17.1/` — take `terraform_1.17.1_windows_amd64.zip` |
| tflint 0.65.0 | `https://github.com/terraform-linters/tflint/releases/tag/v0.65.0` — take `tflint_windows_amd64.zip` |
| trivy 0.76.1 | `https://github.com/aquasecurity/trivy/releases/tag/v0.76.1` — take the Windows 64-bit archive |

Create one folder, for example `C:\tools\bin`, put all three `.exe` files in
it, then add it to `PATH`:

```powershell
[Environment]::SetEnvironmentVariable(
  "Path", $env:Path + ";C:\tools\bin", "User")
```

Open a new terminal and verify:

```bash
terraform -version   # Terraform v1.17.1
tflint --version     # 0.65.0
trivy --version      # 0.76.1
```

> **Verify the download.** `arifa-infra`'s CI checks a SHA-256 for every tool
> archive before unpacking it, because a release tag can be moved by an
> attacker — trivy's action tags were hijacked in March 2026. The checksums are
> in `.github/workflows/terraform.yml`. Doing the same locally is cheap:
> `sha256sum tflint_windows_amd64.zip`.

## Install on macOS

```bash
brew install tfenv tflint trivy pre-commit awscli
cd arifa-infra
tfenv install        # reads .terraform-version, installs exactly 1.17.1
tfenv use 1.17.1
```

`tfenv` is a version switcher. It reads `.terraform-version` and gives each
repository the version it asks for, which is why macOS users never hit the
version error.

## Install on Linux

```bash
# tfenv
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
exec bash
cd arifa-infra && tfenv install

# tflint
curl -fsSL https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# trivy — see the releases page for the pinned version
```

## Version managers are worth it

Different repositories will want different Terraform versions. Rather than
reinstalling by hand, use a manager that reads `.terraform-version`:

- **tfenv** — the common choice, macOS and Linux
- **asdf** or **mise** — manage Terraform alongside Node, Python, etc.

There is no mature `tfenv` for Windows. The practical Windows approach is one
folder per version plus a `PATH` switch, or run Terraform inside WSL.

## AWS credentials

Terraform does not store credentials. It picks them up the same way the AWS CLI
does, in this order:

1. Environment variables — `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
2. `AWS_PROFILE` and `~/.aws/config`
3. Instance or container roles (how CI authenticates)

`arifa-infra` uses a named SSO profile. Configure it once:

```bash
aws configure sso --profile arifa
#   account 483075907105, your role, region eu-central-1
```

Then, whenever the session expires:

```bash
aws sso login --profile arifa
```

The Makefile exports the profile for you on local runs:

```makefile
ifndef CI
export AWS_PROFILE ?= arifa
endif
```

CI sets `CI=true`, skips this, and uses OIDC instead (chapter 13).

> **Never put credentials in `.tf` files.** Not in a variable, not in a default,
> not "temporarily". They end up in git and in the state file. Chapter 14.

## Guard rails worth adding

**Pin the version per repository.** Commit a `.terraform-version` file and a
`required_version` constraint. Both exist in `arifa-infra`.

**Install the pre-commit hooks**, so the checks run before you can commit:

```bash
pip install pre-commit     # or brew install pre-commit
cd arifa-infra
pre-commit install
```

Now `terraform fmt`, `validate`, `tflint` and `trivy` run on every commit. See
`.pre-commit-config.yaml`.

**Enable shell completion** (bash):

```bash
terraform -install-autocomplete
```

## Confirm the setup

From `arifa-infra`:

```bash
bash scripts/check.sh
```

Success looks like four gates running and exit 0. A missing tool now reports
itself clearly before anything else runs:

```
missing tool: tflint trivy
install the pinned versions listed under Setup in README.md
```

## Try it

1. Get `terraform -version` to print exactly `v1.17.1`.
2. Install tflint and trivy; confirm the versions match the table above.
3. Run `bash scripts/check.sh fmt` from `arifa-infra`. It needs no AWS access
   and should pass.
4. Run `bash scripts/check.sh` and get all four gates green.

## Check yourself

1. What versions does `~> 1.17.0` allow? What about `~> 1.17`?
2. Why does CI verify a SHA-256 before unpacking tflint?
3. Where does Terraform look for AWS credentials, and in what order?
4. Why does the Makefile skip setting `AWS_PROFILE` when `CI` is set?

---

Previous: [01 — What Terraform is](01-what-is-terraform.md) ·
Next: [03 — The HCL language](03-hcl-language.md)
