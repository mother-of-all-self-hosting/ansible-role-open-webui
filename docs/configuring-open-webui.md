<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Thomas Miceli
SPDX-FileCopyrightText: 2024 noah
SPDX-FileCopyrightText: 2024, 2025 MASH project contributors
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Open WebUI

This is an [Ansible](https://www.ansible.com/) role which installs [Open WebUI](https://openwebui.com/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Open WebUI is a self-hosted web interface for AI models, which works with [Ollama](https://ollama.com/) and OpenAI-compatible APIs.

See the project's [documentation](https://docs.openwebui.com/) to learn what Open WebUI does and why it might be useful to you.

## Prerequisites

To run an Open WebUI instance it is necessary to prepare a database. You can use [Postgres](https://www.postgresql.org/) or [SQLite](https://www.sqlite.org/). The SQLite database file will be automatically created by the service if it is enabled.

If you are looking for an Ansible role for Postgres, you can check out [this role (ansible-role-postgres)](https://github.com/mother-of-all-self-hosting/ansible-role-postgres) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

Open WebUI does not run models by itself. To chat with models it needs a connection to Ollama or to an OpenAI-compatible API. If you are looking for an Ansible role for Ollama, you can check out [this role (ansible-role-ollama)](https://github.com/mother-of-all-self-hosting/ansible-role-ollama) maintained by the MASH team.

## Adjusting the playbook configuration

To enable Open WebUI with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# open_webui                                                           #
#                                                                      #
########################################################################

open_webui_enabled: true

########################################################################
#                                                                      #
# /open_webui                                                          #
#                                                                      #
########################################################################
```

### Set the hostname

To enable Open WebUI you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
open_webui_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

**Note**: hosting Open WebUI under a subpath (by configuring the `open_webui_path_prefix` variable) is not possible due to Open WebUI's technical limitations.

**Note**: Open WebUI stores the URL built from the hostname in its database on its first start (see [below](#connecting-to-models)), so if you change the hostname later, change the URL in the admin panel as well.

### Set the secret key

You also need to set a secret key, which Open WebUI uses to sign session tokens and encrypt sensitive data. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `YOUR_SECRET_KEY_HERE` with your own value, generated with `pwgen -s 64 1` or in another way.

```yaml
open_webui_environment_variables_webui_secret_key: YOUR_SECRET_KEY_HERE
```

### Set the administrator's account details (optional)

By default, the first user who signs up becomes the administrator, so anyone who reaches the instance before you can take that role. To have Open WebUI create the administrator's account on its first start instead, add the following configuration to your `vars.yml` file:

```yaml
open_webui_environment_variables_webui_admin_email: ADMIN_EMAIL_ADDRESS_HERE
open_webui_environment_variables_webui_admin_password: ADMIN_PASSWORD_HERE
```

Generating a strong password (e.g. `pwgen -s 64 1`) is recommended for `open_webui_environment_variables_webui_admin_password`.

### Configuring database

#### Specify database

Open WebUI can use either Postgres or SQLite, and you need to choose one.

To use Postgres, add the following configuration to your `vars.yml` file:

```yaml
open_webui_database_type: postgres
```

Set `sqlite` to use SQLite. The SQLite database is stored in the directory specified with `open_webui_data_path`.

For other settings, check variables such as `open_webui_database_*` on [`defaults/main.yml`](../defaults/main.yml).

#### Configuring connection to the database server (optional)

By default the role is configured to establish connection with the Postgres server via the Unix socket. You can mount the Unix socket by adding the following configuration to your `vars.yml` file:

```yaml
# Specify the path to the Postgres Unix socket path on the host (bind-mount source)
open_webui_database_socket_path_host: ""
```

Setting it enables to connect to the Postgres server via Unix socket mounted in the container at `/run-postgres/.s.PGSQL.5432`.

If TCP connection is preferred, connection via the Unix socket can be disabled by adding the following configuration to your `vars.yml` file:

```yaml
# Disable the connection to Postgres server via a Unix socket
open_webui_database_socket_enabled: false

open_webui_database_hostname: YOUR_POSTGRES_SERVER_HOSTNAME_HERE
open_webui_database_port: 5432
```

### Connecting to models

To connect Open WebUI to an Ollama server, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
open_webui_environment_variables_ollama_base_url: http://YOUR_OLLAMA_SERVER_HOSTNAME_HERE:11434

# If the server runs in a container, connect Open WebUI to its container network
open_webui_container_additional_networks_custom:
  - YOUR_OLLAMA_SERVER_CONTAINER_NETWORK_HERE
```

If you run Ollama with [ansible-role-ollama](https://github.com/mother-of-all-self-hosting/ansible-role-ollama), both the container name and the container network are `ollama` by default (`mash-ollama` with the MASH playbook).

**Note**: Open WebUI gives models its built-in tools in chats, so a model has to support tool calling, or Ollama refuses the chat with an error like `does not support tools`. Ollama lists `tools` among a model's capabilities in the output of `ollama show MODEL_NAME` if it does. To chat with a model which does not, turn off built-in tools for it on the admin panel.

To connect it to an OpenAI-compatible API, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
open_webui_environment_variables_openai_api_base_url: https://api.openai.com/v1
open_webui_environment_variables_openai_api_key: YOUR_API_KEY_HERE
```

>[!NOTE]
> Open WebUI reads these and many other settings from the environment only on its first start, and keeps them in its database afterwards. To change them later, use the admin panel, or set `open_webui_environment_variables_enable_persistent_config` to `false` so that the values in your `vars.yml` file are always used.

### Using the Playwright web loader (optional)

Open WebUI cannot install a browser for its Playwright web loader inside the container, as the container does not run as `root`. To use the loader, run a separate Playwright server and point Open WebUI to it by adding the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
open_webui_environment_variables_additional_variables: |
  WEB_LOADER_ENGINE=playwright
  PLAYWRIGHT_WS_URL=ws://YOUR_PLAYWRIGHT_SERVER_HOSTNAME_HERE:3000
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `open_webui_environment_variables_additional_variables` variable

See [this page](https://docs.openwebui.com/reference/env-configuration) for a complete list of Open WebUI's config options that you can put in `open_webui_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, the Open WebUI instance becomes available at the URL specified with `open_webui_hostname`. With the configuration above, the service is hosted at `https://example.com`.

To get started, open the URL with a web browser, and create a user or log in with the administrator's account set above. The first user becomes the administrator, and Open WebUI then turns sign-up off. To let other users sign up, turn it on again in the admin panel. They need to be approved by the administrator by default.

**Note**: on its first start, Open WebUI downloads the model it uses to search documents into its data directory, which takes a while.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu open-webui` (or how you/your playbook named the service, e.g. `mash-open-webui`).

#### Increase logging verbosity

If you want to increase the verbosity, add the following configuration to your `vars.yml` file:

```yaml
open_webui_environment_variables_global_log_level: DEBUG
```
