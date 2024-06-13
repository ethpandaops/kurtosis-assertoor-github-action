# Kurtosis-Assertoor Github Action

`kurtosis-assertoor-github-action` is a comprehensive GitHub Action that encapsulates both the setup of a Kurtosis Ethereum Testnet environment and the execution of tests via Assertoor. This action automates the process of configuring Kurtosis with your preferred backend (Docker, Kubernetes, or Cloud) and deploying an Ethereum network for testing. Subsequently, it leverages the `assertoor-github-action` to poll the Assertoor API, facilitating the monitoring of test executions on the Ethereum Testnet.

The action runs the Ethereum Kurtosis package from [ethpandaops/ethereum-package](https://github.com/ethpandaops/ethereum-package), allowing for the deployment of a customizable Ethereum Testnet. To tailor the testnet configuration, such as selecting specific client pairs and configuring Assertoor tests, the `ethereum_package_args` input is used to provide a configuration file.

## Features

- **Kurtosis Setup and Configuration:** Automatically installs and configures Kurtosis with the specified version and backend. Supports Docker, Kubernetes, and Cloud setups.
- **Ethereum Testnet Deployment:** Utilizes the Ethereum Kurtosis package to deploy a multi-client Ethereum network, configurable to your testing requirements.
- **Flexible Assertoor Test Execution:** Offers the option to wait for Assertoor test completion and return results, or enabling subsequent actions to perform additional tasks within the testnet environment.
- **Automated Cleanup:** Executes testnet cleanup as a post-action step, allowing the testnet to remain available for subsequent actions if needed.

## Action Behavior

This GitHub Action automates a comprehensive series of steps to set up a complete Ethereum testnet environment using Kurtosis and to run and monitor tests with Assertoor. Below are the detailed tasks performed by the action:

1. **Setup Kurtosis CLI:** Installs the Kurtosis command-line interface (CLI) tool to ensure it's available for the subsequent steps.

2. **Configure Kurtosis Backend:** Configures the necessary backend for Kurtosis based on the `kurtosis_backend` input. If the chosen backend requires additional dependencies, the action handles their installation.

3. **Run Kurtosis with Ethereum Package:** Executes Kurtosis using the Ethereum package, alongside any provided arguments (`ethereum_package_args`). This step deploys an Ethereum testnet, with the package handling the generation of network genesis and the setup of all specified client pairs. It supports all available consensus and execution clients, offering flexibility in the creation of the testnet environment. Additionally, it allows for customization of network parameters such as the fork schedule or slot time, enabling a tailored testing setup that meets specific needs or scenarios.

4. **Extract Assertoor API URL:** If Assertoor is configured to run via the supplied package arguments, the action extracts the Assertoor API URL for test status polling.

5. **Wait for Assertoor Tests to Complete:** If Assertoor is included in the package args and the action is comfigured to wait for assertoor test completion, the action continuously polls the Assertoor API and prints Assertoor logs for live monitoring & status tracking.

6. **Generate Enclave Dump:** If `enclave_dump` is set to `true`, generates a dump of the Kurtosis enclave. This dump is uploaded as a run artifact named `enclave-dump-{enclave_name}`, where `{enclave_name}` refers to either the supplied name or defaults to `gh-{workflow.run_id}`.

7. **Return Assertoor Test Result:** Returns the Assertoor test results upon completion. If any Assertoor test fails, the action also fails, signaling a potential issue. This step is skipped if Assertoor is not configured to run or the action is configured to not wait for assertoor.

8. **Cleanup Kurtosis Enclave:** Regardless of the success or failure of previous steps, this final step ensures the Kurtosis enclave is properly stopped and removed, ensuring no resources are left running unnecessarily. This cleanup process runs to ensure the test environment is properly dismantled after action execution.

By detailing each step, the action ensures users have a clear understanding of the automated processes involved in setting up, executing, and concluding their Ethereum testnet environment and tests.

**Note on Failure Handling:** As defined in Step 7, if any Assertoor test returns a failure result, the action will fail. For subsequent steps in your workflow to process this failure result (leveraging the `test_overview` & `failed_test_details` outputs), those steps must be configured with conditional execution policies such as `if: ${{ failure() }}`, `if: ${{ always() }}`, etc. Without these conditions, GitHub Actions may not execute the steps intended to handle or report on the failure state.

## Inputs

All inputs for this GitHub Action are optional, allowing for flexible configuration according to your specific testing needs. However, for the action to function as intended, especially for customizing the Ethereum testnet environment and Assertoor tests, it's recommended to supply a configuration file via `ethereum_package_args`, as detailed in the Configuration Example section.

- `kurtosis_version`: Specific version of Kurtosis to install. If not specified, defaults to the latest version available.
- `kurtosis_extra_args`: Extra arguments to be passed to the Kurtosis run command.
- `kurtosis_backend`: Backend to use for Kurtosis (options: `docker`, `kubernetes`, `cloud`). Defaults to `docker`.
- `kurtosis_cloud_api_key`: The API key for your Cloud Kurtosis Account, required if using the cloud backend.
- `kurtosis_cloud_instance_id`: The instance id for the cloud Kurtosis box, required if using the cloud backend.
- `kubernetes_config`: The base64 encoded Kubernetes cluster config, required if using the Kubernetes backend.
- `kubernetes_cluster`: The Kubernetes cluster name to use, required if using the Kubernetes backend.
- `kubernetes_storage_class`: The Kubernetes storage class to use, required if using the Kubernetes backend.
- `ethereum_package_branch`: Branch of the ethereum Kurtosis package to use. Defaults to `main`.
- `ethereum_package_args`: Path to the ethereum package arguments file. This is crucial for customizing the testnet and Assertoor tests as described in the Configuration Example.
- `enclave_name`: Kurtosis enclave name to use. Defaults to an auto-generated name based on the workflow run id.
- `enclave_dump`: Whether to generate and upload an enclave dump after test execution (`true`/`false`). Defaults to `true`.

Note: While all settings are optional, not providing specific configurations, especially the `ethereum_package_args`, may result in a generic test environment that might not align with your testing requirements.

## Configuration Example

To customize the Ethereum testnet and Assertoor tests, supply a configuration like the following via `ethereum_package_args`:

```yaml
participants:
  - el_type: geth
    cl_type: teku
  - el_type: nethermind
    cl_type: prysm
  - el_type: erigon
    cl_type: nimbus
  - el_type: besu
    cl_type: lighthouse
  - el_type: reth
    cl_type: lodestar
  - el_type: nimbus
    cl_type: grandine
additional_services:
  - assertoor
assertoor_params:
  run_stability_check: false
  run_block_proposal_check: true
  tests: # optional addtional tests from external urls
    - https://raw.githubusercontent.com/ethpandaops/assertoor-test/master/assertoor-tests/all-opcodes-test.yaml
```

This configuration specifies the Ethereum client pairs for the testnet and configures the Assertoor tests to be executed.

## Outputs

This action provides outputs for use in subsequent steps of your workflow, making it easier to integrate and act upon the test environment setup and test results:

- `services`: A collection of available service URLs as a JSON string. This output maps all ports exported by services to corresponding URLs that can be accessed from subsequent tasks. For example:

- `test_result`: Assertoor Test result. Indicates the overall success or failure of the Assertoor tests.

- `test_overview`: Overview of Assertoor Test results.

- `failed_test_details`: Details of any failed Assertoor Tests.

```json
{
  "assertoor": {
    "http": {
      "url": "http://127.0.0.1:32885",
      "desc": "8080/tcp"
    }
  }
  ...
}
```

This JSON structure provides a convenient way to access services like Assertoor, detailing the URLs and descriptions for each service's exposed ports, facilitating easy integration with other actions or steps in your workflow.

## Usage

Below is an example of how to use the `kurtosis-assertoor-github-action` in your GitHub Actions workflow:

```yaml
jobs:
  ethereum-testnet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup kurtosis testnet and run assertoor tests
        uses: ethpandaops/kurtosis-assertoor-github-action@v1
        with:
          ethereum_package_args: 'path/to/your/config.yml'
          # Additional configurations as needed
```

## Downstream Repositories

This GitHub Action leverages a variety of tools and resources designed to facilitate Ethereum network testing and development. Below is a list of related repositories that are used directly or indirectly by this action, along with descriptions of their purpose and utility:

- **Kurtosis** ([kurtosis-tech/kurtosis](https://github.com/kurtosis-tech/kurtosis)): A platform that simplifies the packaging and launching of ephemeral backend stacks, making advanced testnet setups more accessible to developers.

- **Ethereum Package** ([ethpandaops/ethereum-package](https://github.com/ethpandaops/ethereum-package)): A specialized Kurtosis package that enables the deployment of a private, portable, and modular Ethereum development network, supporting comprehensive testing environments.

- **Assertoor** ([ethpandaops/assertoor](https://github.com/ethpandaops/assertoor)): An Ethereum Testnet Testing Tool that allows for the execution and monitoring of various test scenarios on Ethereum testnets, focusing on network stability and beacon chain actions.

- **Assertoor Github Action** ([ethpandaops/assertoor-github-action](https://github.com/ethpandaops/assertoor-github-action)): A reusable GitHub Action designed to poll the Assertoor API for test status, integrating seamlessly into CI/CD pipelines for Ethereum project development.

For developers interested in creating custom Assertoor tests or seeking inspiration from existing test scenarios, the following repository provides a valuable collection of reference tests:

- **Assertoor Tests Collection** ([ethpandaops/assertoor-test](https://github.com/ethpandaops/assertoor-test/tree/master/assertoor-tests)): A repository containing a variety of Assertoor test playbooks, demonstrating the tool's capabilities and offering a starting point for custom test development.

These resources represent key components of a comprehensive toolchain for Ethereum network testing and development, facilitating a wide range of testing scenarios from individual smart contract interactions to full-scale network stability assessments.

## Contributing

Contributions to `kurtosis-assertoor-github-action` are welcome. Please refer to the project's issues and pull request sections for contributing guidelines.

## License

Refer to the repository's license file for information on the licensing of this GitHub Action and the associated software.

---

**Note:** This README serves as a basic guide. For comprehensive details about Kurtosis, Assertoor, and their integration into your development workflow, please consult their respective documentation.
