# dbt-loom
![pypi version shield](https://img.shields.io/pypi/v/dbt-loom)


A dbt-core plugin to support multi-project deployments.

:warning: **This package depends on dbt-core's plugin functionality, which is still in beta. Please note that this may 
break in the future as dbt Labs solidifies the dbt plugin API.**

## Getting Started

To being, install the `dbt-loom` python package.

```console
pip install dbt-loom
```

Next, create a `dbt-loom` configuration file. This configuration file provides the paths for your
upstream project's manifest files.

```yaml
manifests:
  - path: path/to/manifest.json
    type: file
```

By default, `dbt-loom` will look for `dbt-loom.config.yml` in your working directory. You can also set the 
`DBT_LOOM_CONFIG_PATH` environment variable. In future versions, you will be able to set a variable in your 
dbt_project.yml file instead.

## How does it work?

As of dbt-core 1.6.0-b8, there now exists a `dbtPlugin` class which defines functions that can
be called by dbt-core's `PluginManger`. During different parts of the dbt-core lifecycle (such as graph linking and 
manifest writing), the `PluginManger` will be called and all plugins registered with the appriate hook will be executed.

dbt-loom implements a `get_nodes` hook, and uses a configuration file to parse manifests, identify public models, and
inject those public models when called by `dbt-core`. 

## Known Caveats
Cross-project dependencies are a relatively new development, and dbt-core plugins
are still in beta. As such there are a number of caveats to be aware of when using
this tool.

1. dbt plugins are only support in dbt-core version 1.6.0-b8 and newer. This means you must be using a dbt adapter 
compatible with this version.
2. `PluginNodeArgs` are not fully-realized dbt `ManifestNode`s, so documentation generated by `dbt docs generate` may 
be sparse when viewing injected models. 
