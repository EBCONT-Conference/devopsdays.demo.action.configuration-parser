# Action configuration parser

## About

This GitHub action can be used to select the correct environment based on the current GitHub context as well the provided YAML configuration.
This repository provides multiple YAML schemas which can be used to create deployment configurations.
This action will parse these configuration files and will provide adequate outputs into the GitHub environment and output.

## Contacts

- Owner: EBCONT operations
- Support: [EBCONT operations](https://support-op.ebcont.com)

## Compatibility

This action **_does not run_** a checkout but a checkout is required! Therefore, ensure your workflow executes a full
checkout prior due running the action itself. Additonally, this action uses the latest version of Python 3 and only works
on `Ubuntu`! Any other OS might not work as intended.

## Schemas

This actions supports the parsing of serval schemas. Under the hood, everything is defined via Pydantic. JSON and Markdown files are available depicting the schemas. You may use a JSON reference within your YAML file to
write and validate your configuration. The Markdown files can be used as documentation.

You may simply add the correct schema validator for your YAML file by adding this line at the top of your YAML file and specifying the correct JSON URL:

````yaml
# yaml-language-server: $schema=
````

### Base

The base schema is not used directly for configuration and therefore cannot be parsed on its own. However, it provides the foundation for all other schemas.

It defines common properties such as triggers, events, environments, runner, rootless, auto options, and more. These settings are then available across all other schemas.

### Deployment

Deployment schemas are used to define deployment configurations such as `Helm`, `SSH`, or `Terraform`. They determine the target environment based on the current context and also describe the deployment configuration itself.

#### HELM

The HELM schema can be used to configure HELM deployments for Kubernetes clusters. It supports dispatch configuration, which allows it to be integrated cleanly into split CI/CD repositories.

In addition to basic triggers, you can define the namespace and the name of the chart to install. You can also specify the HELM version to use and whether the job should run inside a container. If needed, you can disable chart linting.

By default, the resulting workflow only runs for an environment when the chart is meant to be deployed. This ensures that deployment gates are triggered only for actual deployments. Linting alone does not trigger deployment gates.

You can change the chart path, which defaults to `helm/`. However, this setting can only be changed globally, not per environment. As a result, the same chart must be reused across all environments.

References:

- [JSON](.github/schemas/helm.json)
- [Markdown](.github/schemas/helm.md)

Configuration example:

<details>
<summary>Example</summary>

```yaml
runner: example-private-runner
environments:
  dev:
    namespace: example-namespace-dev
    triggers:
      push:
        branches:
          - fix/*
          - feat/*
  stage:
    namespace: example-namespace-stage
    triggers:
      push:
        branches:
          - main
  prod:
    namespace: example-namespace-prod
    triggers:
      push:
        tags:
          - v*
```

</details>

#### SSH

The SSH schema can be used to configure SSH deployments. It supports dispatch configuration, which allows it to be integrated cleanly into split CI/CD repositories.

This schema is similar to the HELM schema. However, instead of defining namespaces and names, you define the shell, user, host, and port used during deployment. The SSH workflow supports both remote and local script execution with the following shells:

- PWSH
- Bash
- SH
- Python

The recommended approach is to use this schema with `localhost` as the host. In that case, the workflow executes the appropriate script directly on the runner, usually from `scripts/main.{ext}` such as `scripts/main.py`. You can also run the workflow in a container.

Because scripts can execute arbitrary code, this workflow is intentionally kept as simple as possible. The recommended setup is a single script file executed locally rather than a more complex script structure. Managing runners is usually easier than managing SSH users and keys.

If you need to execute code remotely over SSH, only single script files are supported and dependencies are not. You can define key-value pairs in the SSH configuration, which are exported as environment variables for the script at runtime. This also supports variable expansion such as `REF_NAME=$GITHUB_REF_NAME`, which sets `REF_NAME` to the resolved value of `$GITHUB_REF_NAME`.

Unlike the HELM schema, the script path and even the shell can be changed per environment. This makes the schema useful for supporting legacy workflows. For modern setups, however, a single script remains the recommended approach.

In general, the SSH workflow is best suited for legacy deployments. HELM remains the more production-ready and enterprise-oriented option because its definitions are cleaner and easier to manage.

References:

- [JSON](.github/schemas/ssh.json)
- [Markdown](.github/schemas/ssh.md)

Configuration example:

<details>
<summary>Example</summary>

```yaml
TBD
```

</details>

#### Terraform

The Terraform schema can be used to configure Terraform deployments. Terraform is somewhat special because it combines both CI- and CD-related tasks. Like the other schemas, it still inherits from the base schema.

One important setting is the `hash` variable. In older Terraform workflows, `hash` was set to `false`. However, newer repositories created through self-service use `true` by default. This setting significantly changes how state is handled in AWS.

Changing this value makes the existing state unusable and causes a new state to be created. Because the default has changed from `false` to `true`, legacy repositories must migrate this setting carefully.

The schema itself does not introduce much beyond that. However, keep in mind that the Terraform workflow also changes depending on the number of environments. If only one environment is configured, the workflow plans less frequently to reduce state locking.

References:

- [JSON](.github/schemas/terraform.json)
- [Markdown](.github/schemas/terraform.md)

Configuration example:

<details>
<summary>Example</summary>

```yaml
TBD
```

</details>

### Build

Build schemas are used to define configurations for build workflows such as `Docker` or `NPM`. 
Unlike deployment schemas, they do not need to specify environments. 
Instead, they describe the exact configuration for one or more builds. As a result, most of these schemas are root schemas by definition.

#### Docker

The Docker schema allows you to define multiple Dockerfiles at the root level. Each entry represents a single build.

At least one Dockerfile configuration must be provided.

References:

- [JSON](.github/schemas/docker.json)
- [Markdown](.github/schemas/docker.md)

Configuration example:

<details>
<summary>Example</summary>

```yaml
TBD
```

</details>

## Usage

### Inputs

| Input                       | Description                                                                                               | Required   | Default      |
|-----------------------------|-----------------------------------------------------------------------------------------------------------|------------|--------------|
| `schema`                    | The schema which should be parsed                                                                         | ✅         |              |
| `deploy`                    | The manual deployment input value. Setting this value will overwrite this value in the YAML configuration | ❌         |              |
| `path`                      | The path to the configuration                                                                             | ❌         |              |
| `environment`               | The environment which can be set to overwrite the environment determination process                       | ❌         |              |

The input for `path` should only be used for special testing purposes and not for regular changes of the path of the configuration file.
The `environment` input can be used to overwrite the environment decision (e.g. when starting a deployment workflow manually).

### Outputs

| Output          | Description                                                   |
|-----------------|---------------------------------------------------------------|
| `json`          | The output JSON representation of the selected schema         |

You may simply use the JSON output like `${{ fromJSON(steps.environment.outputs.json).environment }}`.
This way, a dynamic output can be created for all schemas without needing to update the action for each output.

### Example

```yml
uses: ebcont/ebcont.it-support.action.environment-selection@v3
with:
  schema: helm
  deploy: ${{ inputs.deploy }}
```
