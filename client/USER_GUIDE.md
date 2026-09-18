# Registration API Client User Guide

`reg-api-client` publishes products to the Registration API Gateway.

It requires:
- the product or STAC file you want to ingest;
- the Registration API collection ID;
- your Registration API username and password;
- the Registration API and S3 endpoint URLs.

The client reads or downloads the product, uploads its assets to the Stage-in
S3 bucket, creates or updates the STAC Item metadata, and posts the
Item to the Registration API using your Data Provider account.

## Quick Start

### With Docker

#### Get Docker

- On macOS and Windows, install Docker Desktop from
  <https://www.docker.com/products/docker-desktop/>.
- On Linux, install Docker Engine from your distribution packages or from the
  Docker documentation at <https://docs.docker.com/engine/install/>.

Start Docker Desktop or the Docker daemon, then verify the installation:

```bash
docker version
docker run --rm hello-world
```

On Linux, follow your local policy if `docker run` requires membership in the
`docker` group or `sudo`.

On Apple Silicon Macs, the published client image may run under AMD64 emulation.
If Docker prints a platform warning, add `--platform linux/amd64` after
`docker run`.

Check that Docker can run `reg-api-client`:

```bash
docker run --rm eoepca/reg-api-client:latest --help
```

Prepare the values you received from the Registration API operator:

```bash
export RAPI_USERNAME="my-username"
export RAPI_PASSWORD="my-password"
export RAPI_COLLECTION_ID="MY_COLLECTION"
#Select your endpoint (defaults to eoresults.esa.int)
export RAPI_ENDPOINT="https://eoresults.esa.int/reg-api/"
export RAPI_S3_ENDPOINT="https://eoresults.esa.int"
```

Windows PowerShell:

```powershell
$env:RAPI_USERNAME = "my-username"
$env:RAPI_PASSWORD = "my-password"
$env:RAPI_COLLECTION_ID = "MY_COLLECTION"
$env:RAPI_ENDPOINT = "https://eoresults.esa.int/reg-api/"
$env:RAPI_S3_ENDPOINT = "https://eoresults.esa.int"
```

Both endpoints must use `http://` or `https://` and belong to the environment
where your user, collection, and stage-in bucket were created. Do not use a
bare hostname.

`RAPI_S3_ENDPOINT` is the stage-in S3 frontend base URL only. Don't append
`/reg-api`, a bucket name, an `/sg-*` path, or any port.

If the operator gave you a stage-in bucket name, set it explicitly:

```bash
export RAPI_STAGEIN_BUCKET="sg-my-username-generic"
```

Windows PowerShell:

```powershell
$env:RAPI_STAGEIN_BUCKET = "sg-my-username-generic"
```

You can test the S3 endpoint and credentials before running the full client by
uploading a small file:

```bash
echo "test" > test-upload.txt
docker run --rm \
  -v "$PWD:/data" \
  -e RAPI_USERNAME \
  -e RAPI_PASSWORD \
  -e RAPI_S3_ENDPOINT \
  -e RAPI_STAGEIN_BUCKET \
  --entrypoint /bin/sh \
  eoepca/reg-api-client:latest \
  -lc 'AWS_ACCESS_KEY_ID="$RAPI_USERNAME" AWS_SECRET_ACCESS_KEY="$RAPI_PASSWORD" /bin/s5cmd --endpoint-url "$RAPI_S3_ENDPOINT" cp /data/test-upload.txt "s3://$RAPI_STAGEIN_BUCKET/test-upload.txt"'
```

Windows PowerShell:

```powershell
"test" | Set-Content -NoNewline test-upload.txt
docker run --rm `
  -v "${PWD}:/data" `
  -e RAPI_USERNAME `
  -e RAPI_PASSWORD `
  -e RAPI_S3_ENDPOINT `
  -e RAPI_STAGEIN_BUCKET `
  --entrypoint /bin/sh `
  eoepca/reg-api-client:latest `
  -lc 'AWS_ACCESS_KEY_ID="$RAPI_USERNAME" AWS_SECRET_ACCESS_KEY="$RAPI_PASSWORD" /bin/s5cmd --endpoint-url "$RAPI_S3_ENDPOINT" cp /data/test-upload.txt "s3://$RAPI_STAGEIN_BUCKET/test-upload.txt"'
```

After a successful check, remove the local file and S3 test object:

```bash
docker run --rm \
  -e RAPI_USERNAME \
  -e RAPI_PASSWORD \
  -e RAPI_S3_ENDPOINT \
  -e RAPI_STAGEIN_BUCKET \
  --entrypoint /bin/sh \
  eoepca/reg-api-client:latest \
  -lc 'AWS_ACCESS_KEY_ID="$RAPI_USERNAME" AWS_SECRET_ACCESS_KEY="$RAPI_PASSWORD" /bin/s5cmd --endpoint-url "$RAPI_S3_ENDPOINT" rm "s3://$RAPI_STAGEIN_BUCKET/test-upload.txt"'
rm -f test-upload.txt
```

Windows PowerShell:

```powershell
docker run --rm `
  -e RAPI_USERNAME `
  -e RAPI_PASSWORD `
  -e RAPI_S3_ENDPOINT `
  -e RAPI_STAGEIN_BUCKET `
  --entrypoint /bin/sh `
  eoepca/reg-api-client:latest `
  -lc 'AWS_ACCESS_KEY_ID="$RAPI_USERNAME" AWS_SECRET_ACCESS_KEY="$RAPI_PASSWORD" /bin/s5cmd --endpoint-url "$RAPI_S3_ENDPOINT" rm "s3://$RAPI_STAGEIN_BUCKET/test-upload.txt"'
Remove-Item -Force .\test-upload.txt
```

To check whether Docker can reach the S3 endpoint:

```bash
docker run --rm \
  --entrypoint curl \
  eoepca/reg-api-client:latest \
  -sS -o /dev/null -w "%{http_code}\n" \
  "$RAPI_S3_ENDPOINT"
```

Windows PowerShell:

```powershell
docker run --rm `
  -e RAPI_S3_ENDPOINT `
  --entrypoint /bin/sh `
  eoepca/reg-api-client:latest `
  -lc 'curl -sS -o /dev/null -w "%{http_code}\n" "$RAPI_S3_ENDPOINT"'
```

Any HTTP code means Docker reached the endpoint. For example, `301` means the
endpoint replied with a redirect, so DNS and network access are working for
this check. `Could not resolve host` means the endpoint hostname is wrong or
not available on your current network.

### Quick Start Locally

Local execution requires:

- Python 3.9 or newer, preferably the current stable Python 3 release from
  <https://www.python.org/downloads/>;
- `curl`;
- `s5cmd`;
- optionally, `rio-stac` if you want the client to extract geospatial metadata
  from supported raster files.

Install `rio-stac` only when using `--assets-rio-stac` to extract metadata from
supported raster files.

#### 1. Install Python

Install Python 3.9 or newer from <https://www.python.org/downloads/> or your
package manager. On Windows, select **Add python.exe to PATH** during
installation. Verify it with:

```bash
python3 --version
python3 -m pip --version
```

Windows:

```powershell
py --version
py -m pip --version
```

If these commands fail on Windows, reopen the terminal and check that Python is
on `PATH`.

#### 2. Create A Working Folder

Choose or create a folder where you will keep the client and the files you want
to publish.

macOS:

```bash
mkdir -p "$HOME/reg-api-client"
cd "$HOME/reg-api-client"
```

Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force "$HOME\reg-api-client"
Set-Location "$HOME\reg-api-client"
```

#### 3. Create A Python Virtual Environment

macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Windows PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

If PowerShell refuses to activate the virtual environment, run this command in
the same PowerShell window and then try the activation command again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

#### 4. Install Local Tools

Install `s5cmd`. The client uses it to upload files to the stage-in S3
bucket.

macOS with Homebrew:

```bash
brew install s5cmd
```

If needed, install Homebrew from <https://brew.sh/> first.

On Windows, download and extract the appropriate archive from the
[latest s5cmd release](https://github.com/peak/s5cmd/releases/latest), then put
`s5cmd.exe` beside `reg-api-client` or on `PATH`.

Check that `s5cmd` works:

```bash
s5cmd version
```

Windows PowerShell:

```powershell
.\s5cmd.exe version
```

`curl` is normally already installed on macOS and recent Windows systems. Check
it with:

```bash
curl --version
```

If you want the client to create geospatial STAC metadata from GeoTIFF files,
install `rio-stac` inside the activated virtual environment:

```bash
python -m pip install rio-stac
```

If `rio-stac` installation fails on your local computer, use the Docker quick
start instead. Docker avoids most local geospatial dependency issues.

#### 5. Download The Client

Download the client script:

```bash
curl -L -o reg-api-client https://github.com/EOEPCA/reg-api/raw/refs/heads/main/client/reg-api-client
chmod +x reg-api-client
```

Windows PowerShell:

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/EOEPCA/reg-api/raw/refs/heads/main/client/reg-api-client" `
  -OutFile "reg-api-client"
```

#### 6. Check The Client

Check that it runs:

```bash
./reg-api-client --help
```

Windows PowerShell:

```powershell
python .\reg-api-client --s5cmd-path .\s5cmd.exe --help
```

#### 7. Set the Reg API Endpoint Values

Use the values provided by the reg-api Gateway operator.

macOS:

```bash
export RAPI_USERNAME="my-username"
export RAPI_PASSWORD="my-password"
export RAPI_COLLECTION_ID="MY_COLLECTION"
#Endpoint of the reg-api as provided by Gateway operator (defaults to eoresults.esa.int)
export RAPI_ENDPOINT="https://eoresults.esa.int/reg-api/"
export RAPI_S3_ENDPOINT="https://eoresults.esa.int"
export RAPI_STAGEIN_BUCKET="sg-my-username-generic"
```

Windows PowerShell:

```powershell
$env:RAPI_USERNAME = "my-username"
$env:RAPI_PASSWORD = "my-password"
$env:RAPI_COLLECTION_ID = "MY_COLLECTION"
$env:RAPI_ENDPOINT = "https://eoresults.esa.int/reg-api/"
$env:RAPI_S3_ENDPOINT = "https://eoresults.esa.int"
$env:RAPI_STAGEIN_BUCKET = "sg-my-username-generic"
```

#### 8. Run A Dry Run

Use `--dry-run -C` first. This checks what STAC metadata the client would send
without uploading or publishing anything.

```bash
./reg-api-client \
  --dry-run -C \
  --assets-rio-stac \
  --rapi-endpoint "$RAPI_ENDPOINT" \
  --rapi-s3-endpoint "$RAPI_S3_ENDPOINT" \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  -b "$RAPI_STAGEIN_BUCKET" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.tif
```

Windows PowerShell:

```powershell
python .\reg-api-client `
  --s5cmd-path .\s5cmd.exe `
  --dry-run -C `
  --assets-rio-stac `
  --rapi-endpoint "$env:RAPI_ENDPOINT" `
  --rapi-s3-endpoint "$env:RAPI_S3_ENDPOINT" `
  -u "$env:RAPI_USERNAME" `
  -p "$env:RAPI_PASSWORD" `
  -c "$env:RAPI_COLLECTION_ID" `
  -b "$env:RAPI_STAGEIN_BUCKET" `
  --item-datetime "2024-01-01T00:00:00Z" `
  .\my-product.tif
```

Replace `my-product.tif` with the path to your own file. For example:

```bash
/Users/alex/Downloads/S2B_MSIL1C_20220328T144749_T28XEP_COG.tif
```

Windows PowerShell:

```powershell
C:\Users\alex\Downloads\S2B_MSIL1C_20220328T144749_T28XEP_COG.tif
```

#### 9. Publish After The Dry Run Succeeds

Run the same command without `--dry-run -C`:

```bash
./reg-api-client \
  --assets-rio-stac \
  --rapi-endpoint "$RAPI_ENDPOINT" \
  --rapi-s3-endpoint "$RAPI_S3_ENDPOINT" \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  -b "$RAPI_STAGEIN_BUCKET" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.tif
```

Windows PowerShell:

```powershell
python .\reg-api-client `
  --s5cmd-path .\s5cmd.exe `
  --assets-rio-stac `
  --rapi-endpoint "$env:RAPI_ENDPOINT" `
  --rapi-s3-endpoint "$env:RAPI_S3_ENDPOINT" `
  -u "$env:RAPI_USERNAME" `
  -p "$env:RAPI_PASSWORD" `
  -c "$env:RAPI_COLLECTION_ID" `
  -b "$env:RAPI_STAGEIN_BUCKET" `
  --item-datetime "2024-01-01T00:00:00Z" `
  .\my-product.tif
```

## Command Usage

Most commands follow this structure:

```bash
reg-api-client [options] PRODUCT
```

The product and most common options are:

| Option                        | Meaning                                                                                                                                                                                                                |
|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `PRODUCT`                     | Local path, HTTP URL, HTTPS URL, S3 URL, STAC Item, STAC Collection, STAC Catalog, FeatureCollection, or single asset.                                                                                                 |
| `-c`, `--rapi-collection-id`  | Collection where the product will be registered.                                                                                                                                                                       |
| `-u`, `--rapi-username`       | Registration API username.                                                                                                                                                                                             |
| `-p`, `--rapi-password`       | Registration API password.                                                                                                                                                                                             |
| `-b`, `--rapi-stagein-bucket` | Stage-in S3 bucket. Defaults to `sg-<username>-generic`. Use the bucket provided by the operator if it is different.                                                                                                   |
| `--rapi-endpoint`             | Registration API endpoint. Defaults to `https://eoresults.esa.int/reg-api/`. For IVV, use the HTTPS frontend `/reg-api/` URL.                                                                                          |
| `--rapi-s3-endpoint`          | S3 endpoint used for stage-in upload. Defaults to `http://eoresults.esa.int`. Use the frontend base URL only, for example `https://eoresults-ivv.eodes.dedyn.io`; don't include `/reg-api`, a bucket path, or `:9000`. |
| `--dry-run`                   | Do not upload assets and do not register the STAC Item.                                                                                                                                                                |
| `-C`, `--output-stac`         | Print the generated or ingested STAC JSON. Useful with `--dry-run`.                                                                                                                                                    |
| `-v`                          | Print more detail. Use `-v -v` for debug output.                                                                                                                                                                       |
| `-s`                          | Print less detail.                                                                                                                                                                                                     |

For local execution, the program name is usually `./reg-api-client`.

For Docker execution, replace `reg-api-client` with:

```bash
docker run --rm -v "$PWD:/data" eoepca/reg-api-client:latest
```

Windows PowerShell:

```powershell
docker run --rm -v "${PWD}:/data" eoepca/reg-api-client:latest
```

## What The REG-API-Client Accepts

The `PRODUCT` input can be one or more of these:

| Input                       | Example                         | What happens                                                            |
|-----------------------------|---------------------------------|-------------------------------------------------------------------------|
| STAC Item JSON              | `./item.json`                   | The client uploads the Item assets and posts the Item.                  |
| STAC Catalog JSON           | `./catalog.json`                | The client follows item, collection, next, and items links recursively. |
| STAC Collection JSON        | `./collection.json`             | The client follows item, collection, next, and items links recursively. |
| STAC FeatureCollection JSON | `./items.json`                  | The client follows each feature self link, then follows next links.     |
| Single asset file           | `./product.tif`                 | The client creates a basic STAC Item for the file.                      |
| Asset directory             | `./product.zarr/`               | By default, the client zips the directory before upload.                |
| HTTP or HTTPS URL           | `https://example.org/item.json` | The client downloads it with `curl`.                                    |
| S3 URL                      | `s3://bucket/path/item.json`    | The client downloads it with `s5cmd`.                                   |

For product ingestion, upload either a product asset, such as a GeoTIFF or a
STAC Item JSON describing the product. Do not upload the Collection
configuration JSON as the product input. Collection JSON is used to request or
define the collection itself; the collection must already exist and your user
must already be authorized to publish into it before you ingest product Items.

If you provide a JSON file for an individual product, it should be a STAC Item:
a GeoJSON `Feature` with STAC fields such as `type`, `stac_version`, `id`,
`properties` and `assets`. A plain GeoJSON geometry file is not enough unless
it is wrapped as a valid STAC Item.

By default, the client uses `--mode autodetect`. It first tries to read the
input as STAC JSON. If that fails, it treats the input as a single asset.

For a normal binary asset such as a GeoTIFF, this message is expected:

```text
does not seem to be a STAC ... Autodetect will consider it an asset
```

It means the client tried to parse the file as JSON first, saw that it is not
JSON, and then continued with single-asset ingestion. It is not a failure if the
summary later says `0 failures`.

Use `--mode stac` when the input must be valid STAC and should fail otherwise:

```bash
./reg-api-client --mode stac -c "$RAPI_COLLECTION_ID" ./item.json
```

Use `--mode asset` when the input should be treated as a single asset even if it
looks like JSON:

```bash
./reg-api-client --mode asset -c "$RAPI_COLLECTION_ID" ./my-product.bin
```

Use `--mode assetmerge` when several input files should become assets of one
STAC Item:

```bash
./reg-api-client \
  --mode assetmerge \
  --item-ID "my-product" \
  --item-datetime "2024-01-01T00:00:00Z" \
  -c "$RAPI_COLLECTION_ID" \
  ./data.tif ./metadata.xml ./quicklook.png
```

## Credentials

You can pass credentials directly:

```bash
./reg-api-client -u "$RAPI_USERNAME" -p "$RAPI_PASSWORD" -c "$RAPI_COLLECTION_ID" ./my-product.tif
```

Windows PowerShell:

```powershell
python .\reg-api-client -u "$env:RAPI_USERNAME" -p "$env:RAPI_PASSWORD" -c "$env:RAPI_COLLECTION_ID" .\my-product.tif
```

Or you can also use environment variables for the Registration API options:

```bash
export RAPI_USERNAME="my-username"
export RAPI_PASSWORD="my-password"
export RAPI_COLLECTION_ID="MY_COLLECTION"
export RAPI_STAGEIN_BUCKET="sg-my-username-generic"
export RAPI_ENDPOINT="https://eoresults.esa.int/reg-api/"
export RAPI_S3_ENDPOINT="https://eoresults.esa.int"
```

Windows PowerShell:

```powershell
$env:RAPI_USERNAME = "my-username"
$env:RAPI_PASSWORD = "my-password"
$env:RAPI_COLLECTION_ID = "MY_COLLECTION"
$env:RAPI_STAGEIN_BUCKET = "sg-my-username-generic"
$env:RAPI_ENDPOINT = "https://eoresults.esa.int/reg-api/"
$env:RAPI_S3_ENDPOINT = "https://eoresults.esa.int"
```

When the environment variables are set, you can omit those command line options:

```bash
./reg-api-client ./my-product.tif
```

Windows PowerShell:

```powershell
python .\reg-api-client .\my-product.tif
```

Do not paste real passwords into shared scripts, tickets, or documentation.

## Publishing A STAC Item

If you already have a STAC Item JSON file, publish it like this:

```bash
./reg-api-client \
  --dry-run -C \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  ./item.json
```

Check the output. If it looks correct, publish it:

```bash
./reg-api-client \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  ./item.json
```

The STAC Item should contain:

| Field                       | Requirement                                                                                        |
|-----------------------------|----------------------------------------------------------------------------------------------------|
| `type`                      | Must be `Feature`.                                                                                 |
| `stac_version`              | Recommended value is `1.0.0`.                                                                      |
| `id`                        | Must be unique inside the collection and use only letters, numbers, `.`, `_`, and `-`.             |
| `properties.start_datetime` | Required UTC start time.                                                                           |
| `properties.end_datetime`   | Required UTC end time.                                                                             |
| `properties.datetime`       | Optional. If missing, the API uses `start_datetime`.                                               |
| `assets`                    | Required. At least one asset must have role `data` or `documentation`.                             |
| `assets[].href`             | Asset location. Local or relative paths are uploaded through stage-in. Full URLs are kept as URLs. |
| `assets[].type`             | Required media type.                                                                               |
| `assets[].roles`            | Required list of roles.                                                                            |
| `assets[].file:size`        | Required size in bytes. The client can add or check it for uploaded local assets.                  |

The client also makes sure the STAC File extension is listed in
`stac_extensions`.

### Date Format Warning

Use UTC RFC 3339 timestamps in this exact form:

```text
YYYY-MM-DDTHH:MM:SSZ
```

Good examples:

```text
2024-01-01T00:00:00Z
2024-01-31T23:59:59Z
```

Avoid these forms:

```text
2024-01-01 00:00:00
2024-01-01
2024-13-01T00:00:00Z
2024-01-01T00:00:00
```

The client validates values passed with `--item-datetime` and datetime fields
already present in STAC Items before it uploads assets or posts metadata. This
validation also runs in `--dry-run -C`, so malformed timestamps fail locally
with an `Invalid datetime` message and a `failure_reason` in the JSON output.

Before publishing, still inspect the `--dry-run -C` output and confirm that
`properties.datetime`, `properties.start_datetime`, and
`properties.end_datetime` contain the intended dates.

## Publishing A Single Asset

For a single file, the client can create a simple STAC Item for you:

```bash
./reg-api-client \
  --dry-run -C \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.tif
```

For a time range, separate start and end time with `/`:

```bash
./reg-api-client \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-datetime "2024-01-01T00:00:00Z/2024-01-31T23:59:59Z" \
  ./my-product.tif
```

The generated Item ID comes from the filename without the final extension. For
example, `my-product.tif` becomes Item ID `my-product`.

Override the Item ID when needed:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-ID "my-clean-product-id" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.tif
```

If the filename contains characters that are not allowed in a STAC Item ID, fix
them with a regex:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-ID-regex '[^a-zA-Z0-9._-]' '_' \
  --item-datetime "2024-01-01T00:00:00Z" \
  './My Product (final).tif'
```

## Extracting Dates From Filenames

Use `--item-date-regex` when the product date is part of the filename.

The regex must use named groups. Start date groups are:

- `sY`: start year, required;
- `sm`: start month;
- `sD`: start day;
- `sH`: start hour;
- `sM`: start minute;
- `sS`: start second.

End date groups are:

- `eY`: end year;
- `em`: end month;
- `eD`: end day;
- `eH`: end hour;
- `eM`: end minute;
- `eS`: end second.

Example filename:

```text
Test_20010205_20230301_data.tif
```

Extract only the start year:

```bash
./reg-api-client \
  --dry-run -C \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-date-regex 'Test_(?P<sY>[0-9]{4})' \
  ./Test_20010205_20230301_data.tif
```

This produces a start date of `2001-01-01T00:00:00Z`.

Extract start year, month, and day, plus end year:

```bash
./reg-api-client \
  --dry-run -C \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-date-regex 'Test_(?P<sY>[0-9]{4})(?P<sm>[0-9]{2})(?P<sD>[0-9]{2})_(?P<eY>[0-9]{4})' \
  ./Test_20010205_20230301_data.tif
```

Always test date regexes with `--dry-run -C` first.

Dates generated with `--item-date-regex` are written by the client in the
recommended `YYYY-MM-DDTHH:MM:SSZ` form. Still inspect the dry-run output to
confirm the extracted dates are the dates you intended.

## Checking Whether The Item ID Already Exists

Reg-api item IDs must be unique inside a collection. If you ingest the same filename
twice, or reuse the same `--item-ID`, the catalogue rejects the second publication.
The Registration API reports this as:

```text
Item already exists
```

This can be confusing because the upload may already have copied assets to the
stage-in area before the catalogue rejects the STAC Item.

For asset inputs, the default Item ID is the filename without the final
extension. For example:

```text
GeogToWGS84GeoKey5.tif -> GeogToWGS84GeoKey5
```

For STAC inputs, the Item ID is the `id` field in the STAC JSON unless you
override it with `--item-ID` or `--item-ID-regex`.

If you know the STAC catalogue endpoint, check the item before publishing:

```bash
export STAC_ENDPOINT="https://eoresults.esa.int/stac"
export ITEM_ID="GeogToWGS84GeoKey5"

echo "$STAC_ENDPOINT"
echo "$RAPI_COLLECTION_ID"

curl -sS -o /dev/null -w "%{http_code}\n" \
  "${STAC_ENDPOINT%/}/collections/$RAPI_COLLECTION_ID/items/$ITEM_ID"
```

Read the result as:

| HTTP code      | Meaning                                                                                                                                                |
|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `200`          | The Item ID already exists in that collection. Choose a different ID or ask the reg-api endpoint operator which replacement workflow to use.                        |
| `404`          | The Item ID was not found. It should be available for a new publication. This does not mean the product file format is wrong.                          |
| `401` or `403` | You are not authorized to check that catalogue path, or the endpoint needs credentials.                                                                |
| `000`          | Curl did not receive an HTTP response. Check that `STAC_ENDPOINT` is set, starts with `https://`, has no typo, and is reachable from your network. |

If you get `000`, test the catalogue root first:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" "${STAC_ENDPOINT%/}/"
```

The STAC endpoint needs to be provided by the reg-api Operator, but this is usually something like `/stac`, for example:

```bash
export STAC_ENDPOINT="https://eoresults.esa.int/stac"
```

If the ID already exists, publish with a new ID:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-ID "GeogToWGS84GeoKey5-v2" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./GeogToWGS84GeoKey5.tif
```

Use `--item-ID-regex` when the problem is caused by filename characters rather
than a true duplicate.

## Asset Type, Size And Checksum

The Registration API requires every asset to have a media type and size.

When the client handles a local asset, it can:

- add `file:size` if it is missing;
- check `file:size` if it is already present;
- calculate a SHA2-256 multihash and add `file:checksum` if it is missing;
- check `file:checksum` if it is already present;
- infer `type` for common assets.

For `.tif` files, the default inferred type is:

```text
image/tiff; application=geotiff; profile=cloud-optimized
```

For `.zarr` assets, the default inferred type is:

```text
application/x-zarr; profile=cloud-optimized
```

Set the asset type yourself when needed:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-default-asset-type "application/octet-stream" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.bin
```

Disable checksum generation only if you have a specific reason:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-asset-checksum-disable \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.tif
```

## Using rio-stac For Raster Metadata

If your input is a supported raster file, such as a Cloud Optimized GeoTIFF,
`rio-stac` can create richer STAC metadata.

With Docker:

```bash
docker run --rm \
  -v "$PWD:/data" \
  eoepca/reg-api-client:latest \
  --dry-run -C \
  --assets-rio-stac \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-datetime "2024-01-01T00:00:00Z" \
  /data/my-product.tif
```

Locally, install `rio-stac` first, then use the same `--assets-rio-stac` flag.

If `rio-stac` fails, the client logs the error and falls back to a basic STAC
Item.

## Directories And Zarr Assets

If the input asset is a directory, the client zips it before upload by default.
This is useful for directory-encoded assets such as some Zarr products.

Publish a directory as a ZIP file:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.zarr
```

Keep the directory unzipped:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --item-asset-zipping-disable \
  --item-datetime "2024-01-01T00:00:00Z" \
  ./my-product.zarr
```

## Downloading Inputs From HTTP or S3

You can provide a source URL instead of a local path. The client downloads it,
then uploads and registers it in the reg-api Gateway.

```bash
./reg-api-client \
  --rapi-endpoint "$RAPI_ENDPOINT" \
  --rapi-s3-endpoint "$RAPI_S3_ENDPOINT" \
  -c "$RAPI_COLLECTION_ID" \
  https://example.org/path/item.json
```

If the HTTP source requires Basic authentication:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --assets-http-basic-username "$SOURCE_USERNAME" \
  --assets-http-basic-password "$SOURCE_PASSWORD" \
  https://example.org/path/item.json
```

If the HTTP source requires an authorization header:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --assets-http-authorization-token "Bearer $SOURCE_TOKEN" \
  https://example.org/path/item.json
```

The client can also copy data from a source S3 bucket to the reg-api gateway:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  --assets-aws-access-key-id "$SOURCE_AWS_ACCESS_KEY_ID" \
  --assets-aws-secret-access-key "$SOURCE_AWS_SECRET_ACCESS_KEY" \
  --assets-aws-region "eu-central-1" \
  --assets-aws-endpoint "https://s3.example.org" \
  s3://source-bucket/path/item.json
```

You can also provide source S3 credentials through standard AWS environment
variables:

```bash
export AWS_ACCESS_KEY_ID="source-access-key"
export AWS_SECRET_ACCESS_KEY="source-secret-key"
```

These AWS credentials are for downloading the source asset. They are separate
from your Registration API username and password, which are used for uploading
to the Registration API stage-in bucket.

## Multiple Products

You can pass more than one product in the same command:

```bash
./reg-api-client \
  -c "$RAPI_COLLECTION_ID" \
  ./item-1.json ./item-2.json ./item-3.json
```

By default, the client stops after the first failed product.

Use `--continue-on-error` to keep processing later products:

```bash
./reg-api-client \
  --continue-on-error \
  -c "$RAPI_COLLECTION_ID" \
  ./item-1.json ./item-2.json ./item-3.json
```

The exit code is the number of failed products. If `-C` is used and any product
failed, the exit code is `2`.

## CWL And OGC Application Package Usage

The repository includes `reg-api-client.cwl` for CWL-compatible platforms.

Run locally with `cwltool`:

```bash
cwltool reg-api-client.cwl \
  --collection "$RAPI_COLLECTION_ID" \
  --username "$RAPI_USERNAME" \
  --password "$RAPI_PASSWORD" \
  --product "https://example.org/path/item.json"
```

The CWL file includes default Registration API and S3 endpoints. Edit
`reg-api-client.cwl` before deployment if your platform should use different
endpoints.

## Reading The Output

The client logs progress to the terminal. Important messages include:

```text
items published correctly
failures
failure_reason
```

Treat the final summary as the main status line when there are no preceding
`ERROR` messages:

```text
1 items published correctly. 0 failures.
```

This means the dry run or publication succeeded for one item. If the output also
contains an `ERROR`, especially `Output is not a JSON`, verify the item through
the STAC endpoint and check whether the asset moved from stage-in to the final
datastore. Some server-side failures can prevent the client from parsing the
Registration API response cleanly.

For binary files, this INFO line is normal:

```text
does not seem to be a STAC ... Autodetect will consider it an asset
```

Actual failures are reported with `ERROR` lines and usually include a
`failure_reason` in the JSON output.

Use `-C` to print the generated or returned STAC JSON:

```bash
./reg-api-client --dry-run -C -c "$RAPI_COLLECTION_ID" ./my-product.tif
```

Use `-v` for more information:

```bash
./reg-api-client -v -c "$RAPI_COLLECTION_ID" ./my-product.tif
```

Use `-v -v` for debug-level output:

```bash
./reg-api-client -v -v -c "$RAPI_COLLECTION_ID" ./my-product.tif
```

## Troubleshooting

| Problem                                                                             | What to check                                                                                                                                                                                                                                                                                    |
|-------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `No Data Provider Username provided`                                                | Pass `-u` or set `RAPI_USERNAME`.                                                                                                                                                                                                                                                                |
| `No Data Provider Password provided`                                                | Pass `-p` or set `RAPI_PASSWORD`.                                                                                                                                                                                                                                                                |
| `s5cmd software does not exist or is not an executable`                             | Install `s5cmd`, put it in `PATH`, or pass `--s5cmd-path`. Docker already includes it.                                                                                                                                                                                                           |
| `curl software does not exist or is not an executable`                              | Install `curl`, put it in `PATH`, or pass `--curl-path`. Docker already includes it.                                                                                                                                                                                                             |
| `Input product ... does not exist or is not accessible`                             | Check the local path, Docker volume mount, URL, or file permissions.                                                                                                                                                                                                                             |
| `No Data Provider S3 Stagein Bucket specified. Defaulting to sg-<username>-generic` | This is only safe if that is your real bucket. If the operator gave you a bucket name, pass `-b "$RAPI_STAGEIN_BUCKET"` or set `RAPI_STAGEIN_BUCKET`.                                                                                                                                            |
| Docker cannot access `/home/...`, `/Users/...`, or `C:\Users\...`                   | The container cannot see arbitrary host paths. Mount the host directory, for example `-v "$HOME/Downloads:/data"` on Linux/macOS or `-v "$env:USERPROFILE\Downloads:/data"` in PowerShell, and pass `/data/file.tif` as the product path.                                                        |
| Docker cannot find `/data/my-product.tif`                                           | Make sure you mounted the directory containing the file and used the matching `/data/...` path inside Docker.                                                                                                                                                                                    |
| Docker prints `linux/amd64` versus `linux/arm64/v8`                                 | This is an Apple Silicon platform warning, not a file-path error. The image can still run under emulation. Add `--platform linux/amd64` after `docker run` if needed.                                                                                                                            |
| `does not seem to be a STAC ... Autodetect will consider it an asset`               | This is expected for binary assets such as `.tif` files. It is only a problem if the final summary reports failures.                                                                                                                                                                             |
| Docker endpoint check returns `301`                                                 | The S3 endpoint is reachable and returned a redirect. This is enough for the connectivity check; continue to the upload step, but keep using the exact endpoint value provided by the operator.                                                                                                  |
| `bad value for --endpoint-url ... scheme is missing`                                | `RAPI_S3_ENDPOINT` is missing `http://` or `https://`. Set it to the full URL, for example `export RAPI_S3_ENDPOINT="https://eoresults-ivv.eodes.dedyn.io"` or the exact endpoint provided by the operator.                                                                                      |
| `XAmzContentSHA256Mismatch` during S3 upload                                        | Use the HTTPS frontend S3 endpoint exactly. This can happen when using the HTTP frontend, a localhost nginx path, or another proxy path for signed S3 PUT requests.                                                                                                                              |
| `InvalidAccessKeyId` during S3 upload                                               | The S3 endpoint did not recognize the username used as `AWS_ACCESS_KEY_ID`. Check that `RAPI_USERNAME` is set correctly, that the S3 user exists in the S3 bucket, and that the frontend endpoint points to the expected endpoint deployment as provided by the operator.                                                             |
| `InvalidRequest ... ?uploads subresource` during S3 upload                          | The S3 client attempted multipart upload and the S3-compatible path rejected the multipart initiation. First verify the HTTPS frontend endpoint. If multipart is still unsupported, use a larger `s5cmd --part-size` or a temporary `s5cmd` wrapper until the client exposes a part-size option. |
| Upload stays at `0 B / ...` and ends with `lookup <host>: no such host`             | Docker cannot resolve the S3 endpoint hostname. Check `echo "$RAPI_S3_ENDPOINT"`. Use the S3 endpoint provided by the operator, connect to the required VPN/internal DNS if using an internal environment, or switch to the public endpoint if appropriate.                                  |
| `s5cmd ... returned non-zero exit status 1` during upload                           | Read the `s5cmd` error immediately above it. Common causes are wrong S3 endpoint, DNS failure, wrong bucket, or wrong username/password. Add `-s` to the client command if the progress output is too noisy while debugging.                                                                     |
| Duplicate-ID check returns `000`                                                    | Curl did not reach the STAC API. Set `STAC_ENDPOINT`, for example `https://eoresults.esa.int/stac`, and retry with `curl -sS` so connection errors are printed.                                                                                                                              |
| `No collection specified`                                                           | Pass `-c "$RAPI_COLLECTION_ID"` unless the STAC Item already contains the correct collection.                                                                                                                                                                                                    |
| `datetime` or `start_datetime` missing                                              | Add dates to the STAC Item, pass `--item-datetime`, or pass `--item-date-regex`.                                                                                                                                                                                                                 |
| `Invalid datetime`                                                                  | Use `YYYY-MM-DDTHH:MM:SSZ`, check all three datetime fields in the printed STAC, and avoid spaces, missing `Z`, or invalid month/day values.                                                                                                                                                     |
| `Item should contain at least one asset with 'data' role`                           | Add `roles: ["data"]` to at least one asset, or use asset mode so the client creates it.                                                                                                                                                                                                         |
| `File size does not match STAC metadata`                                            | The asset file changed after the STAC was created. Update `file:size` or let the client generate it.                                                                                                                                                                                             |
| `File checksum does not match STAC metadata`                                        | The asset content changed after the checksum was created. Update `file:checksum` or check that the right file is being used.                                                                                                                                                                     |
| `Item already exists`                                                               | The reg-api catalogue already contains that Item ID in the selected collection. Check `GET /collections/{collection}/items/{id}` on the STAC endpoint, then use a different `--item-ID` or ask the reg-api operator which replacement workflow to use.                                               |
| `User is not authorized to this collection`                                         | Ask the Registration API operator to confirm your username and collection authorization.                                                                                                                                                                                                         |
| `host.docker.internal` or `socket.gaierror`                | Contact the Registration API operator to check user authorization and catalogue address configuration.                                     |
| `no partition of relation "items" found for row`                                    | The collection was not added to the catalogue database. Ask the operator to register the colection and erify the collection through the STAC endpoint before retrying.                                                                                                             |
| `Permission denied ... stagein ... -> ... assets`                                   | The Registration API found the staged asset but cannot move it to the final datastore. Report this to the Registration API operator to fix ownership and write permissions on the stage-in bucket and collection assets/STAC directories.                                                   |
| Manual POST with dry-run output says `Input should be a valid dictionary`           | `--dry-run -C` writes a JSON array. Extract one item before manual POST testing: `jq '.[0]' item.json > item-one.json`.                                                                                                                                                                         |
| Client reports `Output is not a JSON` from reg-api                                  | The Registration API returned an HTML/plain-text error or crashed with a 500. Contact the operator to check what his happening, providing him with a sample of the rrequest.                                     |

### Common Issues

Docker commands use two paths for the same file:

- the host path, before the colon in `-v`;
- the container path, after the colon in `-v`.

The path at the end of the `docker run` command must be the container path, not
the original host path.

For example, if the product is in the current directory, mount the current
directory and use `/data/<filename>`.

Linux, macOS, or Git Bash:

```bash
docker run --rm \
  -v "$PWD:/data" \
  eoepca/reg-api-client:latest \
  --dry-run -C \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-datetime "2024-01-01T00:00:00Z" \
  /data/my-product.tif
```

Windows PowerShell:

```powershell
docker run --rm `
  -v "${PWD}:/data" `
  eoepca/reg-api-client:latest `
  --dry-run -C `
  -u "$env:RAPI_USERNAME" `
  -p "$env:RAPI_PASSWORD" `
  -c "$env:RAPI_COLLECTION_ID" `
  --item-datetime "2024-01-01T00:00:00Z" `
  /data/my-product.tif
```

If the product is in your Downloads directory, mount Downloads and use
`/data/<filename>`.

Linux or macOS:

```bash
docker run --rm \
  -v "$HOME/Downloads:/data" \
  eoepca/reg-api-client:latest \
  --dry-run -C \
  -u "$RAPI_USERNAME" \
  -p "$RAPI_PASSWORD" \
  -c "$RAPI_COLLECTION_ID" \
  --item-datetime "2022-04-24T07:13:48Z" \
  /data/Sentinel-1_20220424T071348_COG.tif
```

Windows PowerShell:

```powershell
docker run --rm `
  -v "$env:USERPROFILE\Downloads:/data" `
  eoepca/reg-api-client:latest `
  --dry-run -C `
  -u "$env:RAPI_USERNAME" `
  -p "$env:RAPI_PASSWORD" `
  -c "$env:RAPI_COLLECTION_ID" `
  --item-datetime "2022-04-24T07:13:48Z" `
  /data/Sentinel-1_20220424T071348_COG.tif
```

Don't use the host path as the product path inside the container:

```bash
# These are wrong unless the same host path was mounted at the same path inside the container.
/home/my-user/Downloads/Sentinel-1_20220424T071348_COG.tif
/Users/my-user/Downloads/Sentinel-1_20220424T071348_COG.tif
C:\Users\my-user\Downloads\Sentinel-1_20220424T071348_COG.tif
```

If you run Docker Desktop, make sure the mounted host directory is allowed in
Docker Desktop file sharing settings. On Apple Silicon Macs, Docker may also
print a `linux/amd64` versus `linux/arm64/v8` platform warning for this image.
That warning is not the file-access error; add `--platform linux/amd64` after
`docker run` if you want to make the emulated platform explicit.
