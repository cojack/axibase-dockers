# Overview

The image contains [Axibase Time Series Database](https://github.com/axibase/atsd) and [Axibase Collector](https://github.com/axibase/axibase-collector) instances. 

The Axibase Collector instance is pre-configured to send data into the local ATSD instance.

The collector instance will automatically initiate the [Docker](https://github.com/axibase/axibase-collector/blob/master/jobs/docker.md) job if the `run` command mounts `/var/run/docker.sock` into the container.

## Image Contents

* [Axibase Time Series Database](https://github.com/axibase/atsd)
* [Axibase Collector](https://github.com/axibase/axibase-collector)
* [collectd](https://github.com/axibase/atsd/tree/master/integration/collectd)

## Launch Container

```sh
docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
  --name=atsd-sandbox \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  axibase/atsd-sandbox:latest
```

> Mounting `/var/run/docker.sock` is necessary for the sandbox to collect performance statistics from the Docker host. Remove the `--volume` option if this is not needed.

Watch the container start log for `All applications started` message.

```sh
docker logs -f atsd-sandbox
```

## Exposed Ports

* Port 8443: ATSD web interface at `https://docker_host:8443/`
* Port 9443: Axibase Collector web interface at `https://docker_host:9443/`.
* Port 8081: ATSD [network commands](https://github.com/axibase/atsd/tree/master/api/network#supported-commands) receiver.

## Default Credentials

By default, new user credentials will be set as follows:

`username: axibase`

`password: axibase`

Open the user account page in ATSD by clicking on the account icon in the upper-right corner of the screen to modify credentials after intial login.

## Container Parameters

| Variable Name | Description |
|------------------|-------------|
| `ATSD_IMPORT_PATH` | Comma-separated paths to files imported into **ATSD**. Path can refer to a file on the mounted file system or to a URL from which the file will be downloaded. |
| `COLLECTOR_IMPORT_PATH` | Comma-separated paths to files imported into **Collector**. Path can refer to a file on the mounted file system or to a URL from which the file will be downloaded. |
| `COLLECTOR_CONFIG` | Specifies parameters to be replaced in Collector configuration files before the import. |
| `SERVER_URL` | URL at which ATSD will be accessible, including schema, hostname, and port, for example: `https://atsd.company_name.com:8443`. |
| `WEBHOOK` | List of incoming webhook templates to be initialized. |
| `SLACK_TOKEN` | Slack bot authentication token. |
| `SLACK_CHANNELS` | Slack channels.|
| `SLACK_CONFIG` | Path to a file with Slack configuration parameters. |
| `TELEGRAM_BOT_TOKEN` | Telegram bot API token. |
| `TELEGRAM_CHAT_ID` | Telegram chat ID.|
| `TELEGRAM_CONFIG` | Path to a file with Telegram configuration parameters. |
| `EMAIL_CONFIG` | Path to a file with Mail Client configuration parameters. |
| `START_COLLECTOR` | Enable or disable **Collector** start |

### Path Formats

This path format is used in `ATSD_IMPORT_PATH`, `COLLECTOR_IMPORT_PATH`, `COLLECTOR_CONFIG` and `EMAIL_CONFIG` variables

1. **URL address** of the file:
   ```sh
   docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
     --name=atsd-sandbox \
     --volume /var/run/docker.sock:/var/run/docker.sock \
     --env ATSD_IMPORT_PATH='https://example.com/atsd-marathon-xml.zip' \
     --env COLLECTOR_IMPORT_PATH='https://example.com/marathon-jobs.xml' \
     axibase/atsd-sandbox:latest
   ```
2. **Absolute path** on the container file system to the file:
   ```sh
   docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
     --name=atsd-sandbox \
     --volume /var/run/docker.sock:/var/run/docker.sock \
     --volume /home/user/atsd-marathon-xml.zip:/atsd-marathon-xml.zip \
     --volume /home/user/marathon-jobs.xml:/marathon-jobs.xml \
     --env ATSD_IMPORT_PATH='/atsd-marathon-xml.zip' \
     --env COLLECTOR_IMPORT_PATH='/marathon-jobs.xml' \
     axibase/atsd-sandbox:latest
   ```
3. **Relative path** to the file. In this case the file should be placed in the `/import` directory on the container file system.
    ```sh
    mkdir /home/user/import
    cp atsd-marathon-xml.zip /home/user/import
    cp marathon-jobs.xml /home/user/import
    docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
      --name=atsd-sandbox \
      --volume /var/run/docker.sock:/var/run/docker.sock \
      --volume /home/user/import:/import \
      --env ATSD_IMPORT_PATH='atsd-marathon-xml.zip' \
      --env COLLECTOR_IMPORT_PATH='marathon-jobs.xml' \
      axibase/atsd-sandbox:latest
   ```

> Note: files or directories mounted into the container, i.e. `--volume /home/user/import:/import`, should not be removed or renamed between container restarts.

### Configuration File Import

#### Import Parameters

`ATSD_IMPORT_PATH` and `COLLECTOR_IMPORT_PATH` variables must be specified using the following format: `path_1,path_2,...,path_N` where each path can refer to either an XML file or zip/tar.gz archive. See [path formats](#path-formats).

#### Variable Substitution

Environment variable substitution will be performed in each of the imported files. Placeholders should have `${ENV.NAME}` format, where `NAME` is the name of the environment variable.

For example, the launch command declares a variable `NAMESPACE` and the imported `jobs.xml` file contains a corresponding placeholder `${ENV.NAMESPACE}`.

```txt
 --env NAMESPACE=Axibase \
 --env COLLECTOR_IMPORT_PATH=jobs.xml \
```

```xml
<attr>${ENV.NAMESPACE}</attr>
```

The imported file will be modified by substituting `${ENV.NAMESPACE}` with the value of the `NAMESPACE` variable:

```xml
<attr>Axibase</attr>
```

To ensure that the XML file remains valid after the variable substitution, wrap the placeholder with `CDATA`.

```xml
<attr><![CDATA[${ENV.NAMESPACE}]]></attr>
```

If no corresponding environment variable is defined for a placeholder specified in the file, the placeholder will be retained 'as is'.


### Server URL

The `SERVER_URL` variable determines the URL, including the protocol, the DNS name and port, at which the database will be accessible.

Usage example:

```sh
docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
  --env SERVER_URL=https://atsd.company_name.com:8443 \
  axibase/atsd-sandbox:latest
```

### Incoming Webhooks

`WEBHOOK` variable specifies a list of templates for creating incoming [webhook](https://github.com/axibase/atsd/blob/master/api/data/messages/webhook.md#sample-urls) URLs.

The list of available templates:

- `aws-cw`
- `github`
- `jenkins`
- `slack`
- `telegram`

Usage example:

```sh
docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
  --name=atsd-sandbox \
  --env SERVER_URL=https://atsd.company_name.com:8443 \
  --env WEBHOOK=github,telegram \
  axibase/atsd-sandbox:latest
```

The generated URLs, including user credentials, will be displayed in the start log:

```txt
...
Webhook user: github
Webhook URL: https://github:9pYV2hxn@atsd.company_name.com:8443/api/v1/messages/webhook/github?exclude=organization.*;repository.*;*.signature;*.payload;*.sha;*.ref;*_at;*.id&include=repository.name;repository.full_name&header.tag.event=X-GitHub-Event&excludeValues=http*&debug=true
...
Webhook user: telegram
Webhook URL: https://telegram:mYz4Peov@atsd.company_name.com:8443/api/v1/messages/webhook/telegram?command.message=message.text
```

### Outgoing Webhooks

Use the following environment variables to configure outgoing webhooks for Slack and Telegram notifications.

| Variable | Description |
|----------|-------------|
| `SLACK_TOKEN` | **Required** Slack bot [authentication token](https://github.com/axibase/atsd/blob/master/rule-engine/notifications/slack.md#add-bot-to-channel). |
| `SLACK_CHANNELS` | Comma-separated list of channels, private groups, or IM channels to send message to. Default value is `general`. |
| `TELEGRAM_BOT_TOKEN` | **Required** Bot API token assigned by [@Botfather](https://telegram.me/BotFather) |
| `TELEGRAM_CHAT_ID` | **Required** Unique identifier for the target chat. |

Usage example:

```sh
docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081
  --name=atsd-sandbox \
  --env SLACK_TOKEN=xoxb-265141031111-QIw323gJUWaX5Fl311111111
  axibase/atsd-sandbox:latest
```

Alternatively, use configuration files to pass these variables into the container using property name format.

`SLACK_CONFIG` and `TELEGRAM_CONFIG` variables specify path to the files with configuration parameters for [Slack](https://github.com/axibase/atsd/blob/master/rule-engine/notifications/slack.md) and [Telegram](https://github.com/axibase/atsd/blob/master/rule-engine/notifications/telegram.md) respectively. File format is the same as for `EMAIL_CONFIG` variable.

* Slack

  Configuration properties for `SLACK_CONFIG`:

  | Property | Variable |
  |----------|----------|
  | `token` | `SLACK_TOKEN` |
  | `channels` | `SLACK_CHANNELS` |

  Contents of `/home/user/import/slack.properties` file.

  ```ls
  token=xoxb-************-************************
  channels=general,devops
  ```

  ```sh
  docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
    --volume /home/user/import:/import
    --name=atsd-sandbox \
    --env SLACK_CONFIG=slack.properties
    axibase/atsd-sandbox:latest
  ```

* Telegram

  Configuration properties for `TELEGRAM_CONFIG`:

  | Property | Variable |
  |----------|----------|
  | `bot_token` | `TELEGRAM_BOT_TOKEN` |
  | `chat_id` | `TELEGRAM_CHAT_ID` |

  ```ls
  bot_token=*********:***********************************
  chat_id=-NNNNNNNNN
  ```

#### Outgoing Webhook Tests

The configured web notifications will be tested on initial start by sending a test message. Test results will be printed in the start log.

* Successful test:

  ```text
  [ATSD] Configure Slack Web Notifications.
  [ATSD]   Slack Web Notification test OK.
  ```

* Sample test messages:

  ![Slack Message Screenshot](resources/slack_message.png)

  ![Telegram Message Screenshot](resources/telegram_message.png)

* Failed test:

  ```text
  [ATSD] Configure Slack Web Notifications.
  [ATSD]   Slack Web Notification test failed.
  [ATSD]   Error: {"ok":false,"error":"invalid_auth"}
  ```

### Mail Client Configuration

`EMAIL_CONFIG` variable specifies the file to read Mail Client configuration from. See [path formats](#path-formats).

The file contains configuration entries in `property_name=value` format.

Supported configuration parameters.

| Property | Description | Default value |
|----------|-------------|---------------|
| `enable` | Enable E-Mail notifications | `on` |
| `server_name` | Server specified in the "From" field, for example `My ATSD Server` | `Axibase TSD` |
| `server` | **Required** Hostname or IP address of your mail server, for example smtp.example.com | - |
| `port` | Mail server port | 587 |
| `sender` | Address specified in the "From" field, for example notify@example.com | Copied from `user` property |
| `user` | **Required** Username of the mailbox user | - |
| `password` | Password of the mailbox user | - |
| `header` | HTML text to add before message body | - |
| `footer` | HTML text to add after message body | - |
| `auth` | Enable authentication | Set to `on` if `password` specified |
| `ssl` | Enable SSL encryption | `on` |
| `upgrade_ssl` | Upgrade an insecure connection to a secure connection using SSL/TLS | `on` |
| `test_email` | E-Mail address to send test message on first start | Copied from `sender` property |

These parameters can be set to `on`/`off` or `true`/`false`: `enable`, `auth`, `ssl`, `upgrade_ssl`.

Sample configuration:

```sh
cat /home/user/import/mail.properties
```

```ls
enabled=true
server_name=ATSD-sandbox
server=mail.example.org
port=587
sender=myuser@example.org
user=myuser@example.org
password=secret
auth=true
ssl=true
upgrade_ssl=true
```

```sh
docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
  --name=atsd-sandbox \
  --volume /home/user/import:/import \
  --env EMAIL_CONFIG=mail.properties \
  axibase/atsd-sandbox:latest
```

### Job Configuration Parameters

`COLLECTOR_CONFIG` is the semicolon-separated sequence of instructions to edit configuration files imported into Collector. 

Each instruction should be specified in the format `file_name.xml:/path/to/properties_file` or `file_name.xml:key1=value1,key2=value2` and will cause the attributes in the XML file to be updated with new values prior to importing the file into Collector.

Instructions can be specified as follows:

1. A path to a file on the container file system (see [path formats](#path-formats])), that contains `key=value` lines:
   ```sh
   docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
     --name=atsd-sandbox \
     --volume /home/user/aws.conf:/aws.conf \
     --env ATSD_IMPORT_PATH='https://github.com/axibase/atsd-use-cases/raw/master/how-to/aws/route53-health-checks/resources/aws-route53-xml.zip' \
     --env COLLECTOR_IMPORT_PATH='https://raw.githubusercontent.com/axibase/atsd-use-cases/master/how-to/aws/route53-health-checks/resources/job_aws_aws-route53.xml' \
     --env COLLECTOR_CONFIG="job_aws_aws-route53.xml:/aws.conf" \
   axibase/atsd-sandbox:latest
   ```
   ```sh
   cat /home/user/aws.conf
   ```
   ```ls
   accessKeyId=key
   secretAccessKey=secret
   ```
2. A key-value pair in `key=value` format:
   ```sh
   docker run -d -p 8443:8443 -p 9443:9443 -p 8081:8081 \
     --name=atsd-sandbox \
     --volume /var/run/docker.sock:/var/run/docker.sock \
     --env ATSD_IMPORT_PATH='https://raw.githubusercontent.com/axibase/atsd-use-cases/master/how-to/marathon/capacity-and-usage/resources/atsd-marathon-xml.zip' \
     --env COLLECTOR_IMPORT_PATH='https://raw.githubusercontent.com/axibase/atsd-use-cases/master/how-to/marathon/capacity-and-usage/resources/marathon-jobs.xml' \
     --env COLLECTOR_CONFIG='marathon-jobs.xml:server=marathon_hostname,port=8080,userName=my-user,password=my-password' \
     axibase/atsd-sandbox:latest
   ```

The XML file update involves replacements of XML tag values, identified with `key`, with new values, for example:

```sh
--env COLLECTOR_CONFIG='marathon-jobs.xml:server=mar1.example.com,userName=netops,password=1234456'
```

- Before
  ```xml
  <server>marathon_hostname</server>
  <port>8080</port>
  <userName>axibase</userName>
  <password></password>
  ```
- After
  ```xml
  <server>mar1.example.com</server>
  <port>8080</port>
  <userName>netops</userName>
  <password>1234456</password>  
  ```

### Collector Start Control

`START_COLLECTOR` variable enables or disables Collector start.

* `on` and `true` values enable start (default).
* `off` and `false` disable start.

```sh
--env COLLECTOR_CONFIG=off
```

This setting applies to the first and all subsequent container starts.

### Parameters Syntax

The variables must not contain whitespace characters.

Semicolons and commas in file names, URLs, key and values must be escaped by `\` character as specified below:

| Variable | Escaping |
|----------|----------|
| `ATSD_IMPORT_PATH` | Only `,` must be escaped. Do not escape `;` |
| `COLLECTOR_IMPORT_PATH` | Only `,` must be escaped. Do not escape `;` |
| `COLLECTOR_CONFIG` | Both `,` and `;` must be escaped |

Variables `WEBHOOK`, `SERVER_URL`, `EMAIL_CONFIG` do not require special escaping for `,` and `;`.

```
--env COLLECTOR_CONFIG='config.xml:password=password\,with\;separators'
```

Additional escaping might be required depending on the shell type and version.
