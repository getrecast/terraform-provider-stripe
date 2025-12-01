# Stripe Terraform Provider

The Stripe Terraform provider uses the official Stripe SDK based on Golang. On top of that, the provider is developed
around the official Stripe API documentation [website](https://stripe.com/docs/api).

The Stripe Terraform Provider documentation can be found on the Terraform Provider documentation [website](https://registry.terraform.io/providers/lukasaron/stripe/latest).

## Usage:
```
terraform {
  required_providers {
    stripe = {
      source = "lukasaron/stripe"
    }
  }
}

provider "stripe" {
  api_key="<api_secret_key>"
}
```

### Environmental variable support

The parameter `api_key` can be omitted when the `STRIPE_API_KEY` environmental variable is present.

---

## Building and Deploying to recast-app

This is the Recast fork of the Stripe Terraform provider with custom modifications. To build and deploy new versions:

### 1. Make Your Changes
- Create a feature branch for your changes
- Make your modifications to the provider code
- Create a PR and merge to `main`

### 2. Tag and Release
```bash
# After PR is merged, checkout main and pull latest
git checkout main
git pull origin main

# Create a new version tag (use semantic versioning with -recast-N suffix)
git tag v3.4.1-recast-2
git push origin v3.4.1-recast-2
```

This triggers the GitHub Actions release workflow which:
- Builds binaries for multiple architectures (darwin_amd64, darwin_arm64, linux_amd64, etc.)
- Creates a GitHub release with all binaries
- Signs the binaries with GPG

### 3. Update recast-app Provider Mirror

Download the new release and update the local mirror in recast-app:

```bash
# Navigate to recast-app terraform directory
cd /path/to/recast-app/terraform

# Download the new release binaries
gh release download v3.4.1-recast-2 --repo getrecast/terraform-provider-stripe \
  --pattern '*darwin*.zip' --pattern '*linux_amd64.zip' --clobber

# Extract binaries
for arch in darwin_amd64 darwin_arm64 linux_amd64; do
  unzip -o terraform-provider-stripe_*_${arch}.zip -d temp_${arch}
done

# Copy to mirror
for arch in darwin_amd64 darwin_arm64 linux_amd64; do
  cp -r temp_${arch}/* \
    terraform-providers-mirror/registry.terraform.io/getrecast/stripe/v3.4.1-recast-2/${arch}/
done

# Clean up
rm -rf temp_* terraform-provider-stripe*.zip

# Remove old provider cache and reinitialize
rm -rf .terraform/providers
terraform init
```

### 4. Provider Configuration

The recast-app uses a filesystem mirror for the custom provider. This is configured via:

- **`.terraformrc`** - Points Terraform to the local mirror directory
- **`.envrc`** - Sets `TF_CLI_CONFIG_FILE` environment variable (for direnv)
- **GitHub Actions** - Uses `TF_CLI_CONFIG_FILE` repository variable

The provider is referenced in Terraform configs as:
```hcl
terraform {
  required_providers {
    stripe = {
      source  = "registry.terraform.io/getrecast/stripe"
      version = "3.4.1-recast-1"
    }
  }
}
```

### 5. State Migration (First Time Only)

If migrating from the upstream `lukasaron/stripe` provider to `getrecast/stripe`, use the migration script:

```bash
cd /path/to/recast-app/terraform
./migrate-stripe-provider.sh <workspace-name>
```

This updates the Terraform state to reference the new provider without recreating resources.

### Important Notes

- **Stripe Price/Product Deletion**: This fork prevents deletion of Stripe prices and products via Terraform, as the Stripe API doesn't support deletion. Instead, set `active = false` to deactivate them.
- **Never tag feature branches**: Always create release tags on the `main` branch after PRs are merged
- **Version naming**: Use format `vX.Y.Z-recast-N` where N increments for each Recast-specific release

---

### Local Debugging
* Build the provider with `go build main.go`
* Move the final binary to the `mv main ~/.terraform.d/plugins/local/lukasaron/stripe/100/[platform]/terraform-provider-stripe_v100` where [platform] is `darwin_arm64` for Mac Apple chip for example.
* Create an HCL code with the following header:
 ```
terraform {
  required_providers {
    stripe = {
      source  = "local/lukasaron/stripe"
      version = "100"
    }
  }
}
```

* Run the solution from the code with the program argument `--debug`
* Copy the `TF_REATTACH_PROVIDERS` value.
* `export TF_REATTACH_PROVIDERS=[value]`
* Put breakpoints in the code
* Remove .terraform folder where the HCL code is.
* Run `terraform init` & `terraform plan` & `terraform apply`