# `homebrew` provider for [`stackql`](https://github.com/stackql/stackql)

This repository is used to generate and document the `homebrew` provider for StackQL, allowing you to query Homebrew packages, formulas, and repositories using SQL-like syntax. The provider is built using the `@stackql/provider-utils` package, which provides tools for converting OpenAPI specifications into StackQL-compatible provider schemas.

## Prerequisites

To use the Homebrew provider with StackQL, you'll need:

1. StackQL CLI installed on your system (see [StackQL](https://github.com/stackql/stackql))
2. Basic understanding of Homebrew API endpoints

## 1. Download the Open API Specification

First, create an OpenAPI specification for the Homebrew API (since Homebrew doesn't provide an official one):

```bash
mkdir -p provider-dev/downloaded
# Create or download the homebrew OpenAPI spec
# You'll need to implement a script that generates an OpenAPI spec from Homebrew's API endpoints
python3 provider-dev/scripts/generate_homebrew_spec.py \
  --output provider-dev/downloaded/openapi.json
```

## 2. Split into Service Specs

Next, split the OpenAPI specification into service-specific files:

```bash
rm -rf provider-dev/source/*
npm run split -- \
  --provider-name homebrew \
  --api-doc provider-dev/downloaded/openapi.json \
  --svc-discriminator tag \
  --output-dir provider-dev/source \
  --overwrite \
  --svc-name-overrides "$(cat <<EOF
{
  "formulas": "formulas",
  "casks": "casks",
  "analytics": "analytics",
  "taps": "taps"
}
EOF
)"
```

## 3. Generate Mappings

Generate the mapping configuration that connects OpenAPI operations to StackQL resources:

```bash
npm run generate-mappings -- \
  --provider-name homebrew \
  --input-dir provider-dev/source \
  --output-dir provider-dev/config
```

Update the resultant `provider-dev/config/all_services.csv` to add the `stackql_resource_name`, `stackql_method_name`, `stackql_verb` values for each operation.

## 4. Generate Provider

This step transforms the split OpenAPI service specs into a fully-functional StackQL provider by applying the resource and method mappings defined in your CSV file.

```bash
rm -rf provider-dev/openapi/*
npm run generate-provider -- \
  --provider-name homebrew \
  --input-dir provider-dev/source \
  --output-dir provider-dev/openapi/src/homebrew \
  --config-path provider-dev/config/all_services.csv \
  --servers '[{"url": "https://formulae.brew.sh/api"}]' \
  --provider-config '{"auth": {"type": "none"}}' \
  --overwrite
```

## 5. Test Provider

### Starting the StackQL Server

Before running tests, start a StackQL server with your provider:

```bash
PROVIDER_REGISTRY_ROOT_DIR="$(pwd)/provider-dev/openapi"
npm run start-server -- --provider homebrew --registry $PROVIDER_REGISTRY_ROOT_DIR
```

### Test Meta Routes

Test all metadata routes (services, resources, methods) in the provider:

```bash
npm run test-meta-routes -- homebrew --verbose
```

When you're done testing, stop the StackQL server:

```bash
npm run stop-server
```

Use this command to view the server status:

```bash
npm run server-status
```

### Run test queries

Run some test queries against the provider using the `stackql shell`:

```bash
PROVIDER_REGISTRY_ROOT_DIR="$(pwd)/provider-dev/openapi"
REG_STR='{"url": "file://'${PROVIDER_REGISTRY_ROOT_DIR}'", "localDocRoot": "'${PROVIDER_REGISTRY_ROOT_DIR}'", "verifyConfig": {"nopVerify": true}}'
./stackql shell --registry="${REG_STR}"
```

Example queries to try:

```sql
-- List all formulas
SELECT 
name,
full_name,
desc,
license,
homepage,
versions,
dependencies,
conflicts_with
FROM homebrew.formulas.formulas;

-- Get information about a specific formula
SELECT 
name,
full_name,
desc,
license,
homepage,
versions,
dependencies,
conflicts_with,
build_dependencies,
requirements
FROM homebrew.formulas.formula
WHERE name = 'wget';

-- List all casks
SELECT 
name,
full_name,
token,
desc,
homepage,
version,
url
FROM homebrew.casks.casks;

-- Get analytics for popular formula installations
SELECT 
formula,
count,
percent
FROM homebrew.analytics.install_30d
ORDER BY count DESC
LIMIT 10;

-- Get analytics for formula installations over longer periods
SELECT 
formula,
count,
percent
FROM homebrew.analytics.install_365d
ORDER BY count DESC
LIMIT 10;
```

## 6. Publish the provider

To publish the provider push the `homebrew` dir to `providers/src` in a feature branch of the [`stackql-provider-registry`](https://github.com/stackql/stackql-provider-registry). Follow the [registry release flow](https://github.com/stackql/stackql-provider-registry/blob/dev/docs/build-and-deployment.md).  

Launch the StackQL shell:

```bash
export DEV_REG="{ \"url\": \"https://registry-dev.stackql.app/providers\" }"
./stackql --registry="${DEV_REG}" shell
```

Pull the latest dev `homebrew` provider:

```sql
registry pull homebrew;
```

Run some test queries to verify the provider works as expected.

## 7. Generate web docs

Provider doc microsites are built using Docusaurus and published using GitHub Pages.  

a. Update `headerContent1.txt` and `headerContent2.txt` accordingly in `provider-dev/docgen/provider-data/`  

b. Update the following in `website/docusaurus.config.js`:  

```js
// Provider configuration - change these for different providers
const providerName = "homebrew";
const providerTitle = "Homebrew Provider";
```

c. Then generate docs using...

```bash
npm run generate-docs -- \
  --provider-name homebrew \
  --provider-dir ./provider-dev/openapi/src/homebrew/v00.00.00000 \
  --output-dir ./website \
  --provider-data-dir ./provider-dev/docgen/provider-data
```  

## 8. Test web docs locally

```bash
cd website
# test build
yarn build

# run local dev server
yarn start
```

## 9. Publish web docs to GitHub Pages

Under __Pages__ in the repository, in the __Build and deployment__ section select __GitHub Actions__ as the __Source__. In Netlify DNS create the following records:

| Source Domain | Record Type  | Target |
|---------------|--------------|--------|
| homebrew-provider.stackql.io | CNAME | stackql.github.io. |

## License

MIT

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.