# Publish Custom Provider in TFC Private Registry

A guide with specific examples of all steps needed to publish a custom provider on the Terraform Cloud private registry. 

HashiCorp's [documentation](https://www.terraform.io/cloud-docs/registry/publish-providers) on publishing custom providers provides detailed steps on the actions related directly to Terraform Cloud but provides only high-level description regarding the additional steps that are not directly related to Terraform Cloud.

The goal is to

* Have a provider named `myprovider`
* Have a version `0.1.0` and a linux/amd64 binary for it. 


## Prerequisites

The guide assumes that commands will be run on MacOS but they should be almost identical on Linux.

* Have the `jq` package installed.

  ```bash
  brew install jq
  ```

* Have the `gnupg` package installed.

  ```bash
  brew install gnupg
  ```

  * Have a pgp keypair that will be used for signing the provider release. 

    * To cerate a new one run the commend below and follow the instructions.

      ```bash
      gpg --full-generate-key
      ```

      The key should use the RSA algorithm. Select other options on your discretion.
   
    * Export the public key

      ```bash
      gpg -o gpg-key.pub -a --export <key id or email>
      ```

## Preparing the provider release

Generally, a user would have the provider binaries for the versions and platforms they want to publish. Here we will use the official `http` provider binary instead. 

### Preparing the binary

* Download a binary for the `http` provider for the Linux/amd64 platform

  ```bash
  curl -o terraform-provider-http_2.1.0_linux_amd64.zip https://releases.hashicorp.com/terraform-provider-http/2.1.0/terraform-provider-http_2.1.0_linux_amd64.zip
  ```

* Expand the archive and remove it

  ```bash
  unzip terraform-provider-http_2.1.0_linux_amd64.zip && rm terraform-provider-http_2.1.0_linux_amd64.zip
  ```

* Rename the extracted binary to match the name expected for the custom provider - `myprovider`

  ```bash
  mv terraform-provider-http_v2.1.0_x5 terraform-provider-myprovider_v0.1.0_x5
  ```

* Create a zip archive with the provider binary

  ```bash
  zip terraform-provider-myprovider_0.1.0_linux_amd64.zip terraform-provider-myprovider_v0.1.0_x5
  ```

### Preparing the provide verification signature

Terraform verifies the integrity of downloaded provider binaries by comparing the shasum of the downloaded binary versus the shasum defined from a signed file downloaded from the registry. To prepare the files related to that:

* Generate a file containing the shasums for the binaries in that version. Since we will be publishing a single platform Linux/amd64 there would be only one binary and only one shasum.

  ```bash
  shasum -a 256 terraform-provider-myprovider_0.1.0_linux_amd64.zip > terraform-provider-myprovider_0.1.0_SHA256SUMS
  ```

* Create a detached signature for the `terraform-provider-myprovider_0.1.0_SHA256SUMS` file using a gpg key

  ```bash
  gpg -sb terraform-provider-myprovider_0.1.0_SHA256SUMS
  ```

  After running this command a file named `terraform-provider-myprovider_0.1.0_SHA256SUMS.sig` will be created.

At this point you should have all the files needed to publish the provider version in the Terraform Cloud private registry.

```bash
terraform-provider-myprovider_0.1.0_SHA256SUMS
terraform-provider-myprovider_0.1.0_SHA256SUMS.sig
terraform-provider-myprovider_0.1.0_linux_amd64.zip
```

### Publishing the provider in the Terraform Cloud registry

Currently, the only available option for publishing a private provider in the Terraform Cloud registry is via the API as described in the Terraform Cloud [documentation](https://www.terraform.io/cloud-docs/registry/publish-providers#publishing-a-provider-and-creating-a-version).

A Terraform Cloud owner user api token is needed to perform the requests.

* Add the public GPG key to the registry via the [API](https://www.terraform.io/cloud-docs/api-docs/private-registry/gpg-keys#add-a-gpg-key).

    * Prepare a payload for the api request e.g. create a file named `gpg-key-payload.json` with content:

      ```json
      {
        "data": {
          "type": "gpg-keys",
          "attributes": {
            "namespace": "<your-tfc-org>",
            "ascii-armor": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n\nmQINB...=txfz\n-----END PGP PUBLIC KEY BLOCK-----\n"
          }   
        }
      }
      ```

         
      * `namespace` is the name of your terraform cloud organization.
      * `ascii-armor` is the exported public gpg key with new lines replaces with literal `\n` characters. To obtain it you can run `sed 's/$/\\n/g' gpg-key.pub | tr -d '\n\r'` assuming that the key is in the `gpg-key.pub` file.
  

    * Make the api call to add the gpg key to the private registry of your Terraform Cloud organization.

        ```bash
        curl -sS \
            --header "Authorization: Bearer $TOKEN" \
            --header "Content-Type: application/vnd.api+json" \
            --request POST \
            --data @gpg-key-payload.json \
            https://app.terraform.io/api/registry/private/v2/gpg-keys | jq '.'
        ```
    
        Note the value of the `"key-id"` property from the response as it will be needed later. A gpg key id consists of the last 8 bytes in hex (last 16 characters) of the key's fingerprint.

  * Create the provider in the Terraform Cloud private registry via the [API](https://www.terraform.io/cloud-docs/api-docs/private-registry/providers#create-a-provider).
    
    *  Prepare a payload for the api request e.g. create a file named `provider-payload.json` with content:
  
    ```json
    {
      "data": {
        "type": "registry-providers",
          "attributes": {
          "name": "myprovider",
          "namespace": "<your-tfc-org>",
          "registry-name": "private"
        }
      }
    }
    ```
      * `name` is the name of the provider.
      * `namespace` is the name of your Terraform Cloud organization.

    * Make the api call to create the provider.

    ```bash
    curl -sS \
      --header "Authorization: Bearer $TOKEN" \
      --header "Content-Type: application/vnd.api+json" \
      --request POST \
      --data @provider-payload.json \
      https://app.terraform.io/api/v2/organizations/<YOUR-TFC-ORG>/registry-providers | jq '.'
    ```

  * Create a version for that provider via the [API](https://www.terraform.io/cloud-docs/api-docs/private-registry/provider-versions-platforms#create-a-provider-version).
    
    * Prepare a payload for the api request e.g. create a file named `provider-version-payload.json` with content:

      ```json
      {
        "data": {
          "type": "registry-provider-versions",
          "attributes": {
            "version": "0.1.0",
            "key-id": "<your-gpg-key-id>",
            "protocols": ["5.0"]
          }
        }
      }
      ```
      * `version` the provider version you want to create.
      * `key-id` the id of a GPG key uploaded to the private registry and used to sign the version shasum's file.
    
    * Make the api call to create the provider version.
  
      ```bash
      curl -sS \
        --header "Authorization: Bearer $TOKEN" \
        --header "Content-Type: application/vnd.api+json" \
        --request POST \
        --data @provider-version-payload.json \
        https://app.terraform.io/api/v2/organizations/<YOUR-TFC-ORG>/registry-providers/private/<YOUR-TFC-ORG>/myprovider/versions | jq '.'
      ```

      Note the URLs in the `data.links.shasums-upload` and `data.links.shasums-sig-upload` properties.

    * Upload the shasums file to the url from the `data.links.shasums-upload` property in the response to the create provider version API call.

      ```bash
      curl -T terraform-provider-myprovider_0.1.0_SHA256SUMS <URL from data.links.shasums-upload>
      ```

    * Upload the shasums signature file to the url from the `data.links.shasums-sig-upload` property in the response to the create provider version API call.

      ```bash
      curl -T terraform-provider-myprovider_0.1.0_SHA256SUMS.sig <URL from data.links.shasums-sig-upload>
      ```

  * Create a Linux/amd64 platform for the version via the [API](https://www.terraform.io/cloud-docs/api-docs/private-registry/provider-versions-platforms#create-a-provider-platform).

    * Prepare a payload for the api request e.g. create a file named `provider-version-payload-platform.json` with content:

      ```json
      {
        "data": {
          "type": "registry-provider-version-platforms",
          "attributes": {
            "os": "linux",
            "arch": "amd64",
            "shasum": "<shasum of the binary archive>",
            "filename": "terraform-provider-myprovider_0.1.0_linux_amd64.zip"
          }
        }
      }
      ```
      * `shasum` the shasum of the archive file containing the binary. Take it from the `terraform-provider-myprovider_0.1.0_SHA256SUMS` file.
      * `filename` the name of the archive file containing the binary. If following the guide it should be the same of the as in the example.
    
    * Make the api call to create the provider version platform.

      ```bash
      curl -sS \
        --header "Authorization: Bearer $TOKEN" \
        --header "Content-Type: application/vnd.api+json" \
        --request POST \
        --data @provider-version-payload-platform.json \
        https://app.terraform.io/api/v2/organizations/<YOUR-TFC-ORG>/registry-providers/private/<YOUR-TFC-ORG>/myprovider/versions/0.1.0/platforms | jq '.'
      ```

      Note the value of the `data.links.provider-binary-upload` property from the response.

    * Upload the archived binary to the `data.links.provider-binary-upload` URL.
  
      ```bash
      curl -T terraform-provider-myprovider_0.1.0_linux_amd64.zip <URL from data.links.provider-binary-upload>
      ```
At this point you there should be a `myprovider` provider published in the private registry for your Terraform Cloud organization. The provider should also have version `0.1.0` released.

![](/screenshots/provider-published.png)

### Verifying if the provider works

To confirm that the provider works

#### Method I

* Create a VCS repository that has a terraform configuration file that requires the custom provider e.g.
  
  ```hcl
  terraform {
    required_providers {
      myprovider = {
        source = "app.terraform.io/<YOUR-TFC-ORG>/myprovider"
        version = "0.1.0"
      }
    }
  }
  ```

* Connect this repository to a workspace within your Terraform Cloud organization
* Perform a run. The plan for that run should be successful and report that no changes are needed as no resources are actually defined. This indicates that the `terraform init` phase passed successfully e.i. that the custom provider was downloaded from the private registry.

![](/screenshots/provider-test-tfc.png)

#### Method II

* Bring up a Linux VM with terraform installed.
* Setup authentication for the Terraform CLI on the VM that allows it to access your Terraform Cloud organization.
* Create a terraform configuration file  that requires the custom provider, same as in the example in [Method I](#method-i)
* Run `terraform init`

The command should be successful and the provider binary should be installed in `./.terraform/providers/app.terraform.io/<YOUR-TFC-ORG>/myprovider/0.1.0/linux_amd64`