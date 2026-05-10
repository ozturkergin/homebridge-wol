<p align="center">
  <img src=".github/banner.png">
</p>

# A Wake on Lan plugin for Homebridge

### Turn your PCs, laptops, servers and more on and off through Siri

<a id="quickstart"></a>

## Quick Start

To install the plugin using a `.tgz` file over SSH, you can use the `hb-service add` command:

```bash
ssh <user>@<host> "sudo hb-service add homebridge-wol-ergin@https://github.com/ozturkergin/homebridge-wol/raw/main/homebridge-wol-ergin-1.0.0.tgz"
```

Add your devices to your `config.json`:

```json
"accessories": [
  {
    "accessory": "NetworkDevice",
    "name": "My MacBook",
    "host": "192.168.1.51",
    "mac": "aa:bb:cc:dd:ee:ff"
  }
]
```

## Table of contents

[Quickstart](#quickstart)<br/>
[Configuration](#configuration)<br />
[Lifecycle](#lifecycle)<br />
[Notes and FAQ](#faq)

<a id="configuration"></a>

## Configuration

To make Homebridge aware of the new plugin, you will have to add it to your configuration usually found in `/root/.homebridge/config.json` or `/home/username/.homebridge/config.json`. If the file does not exist, you can create it following the [config sample](https://github.com/nfarina/homebridge/blob/master/config-sample.json). Somewhere inside that file you should see a key named `accessories`. There you can add configurations like the following.

Note that these are only examples to showcase the different possible configurations. Read through [the configuration section below](#configuration) for detailed information.

```json
"accessories": [
   {
     "accessory": "NetworkDevice",
     "name": "My Macbook",
     "mac": "<mac-address>",
     "host": "macbook.local",
     "pingCommand": "ssh macbook.local 'if [[ $(pmset -g powerstate IODisplayWrangler | tail -1 | cut -c29) -lt 4 ]]; then; exit 1; else; echo 1; fi;'",
     "pingInterval": 45,
     "wakeGraceTime": 10,
     "wakeCommand": "ssh macbook.local caffeinate -u -t 300",
     "shutdownGraceTime": 15,
     "shutdownCommand": "ssh macbook.local sudo shutdown -h now"
   },
   {
     "accessory": "NetworkDevice",
     "name": "My Windows Gaming Rig",
     "mac": "<mac-address>",
     "host": "192.168.1.151",
     "shutdownCommand": "net rpc shutdown --ipaddress 192.168.1.151 --user username%password"
   },
   {
     "accessory": "NetworkDevice",
     "name": "Raspberry Pi",
     "mac": "<mac-address>",
     "host": "192.168.1.251",
     "pingInterval": 45,
     "wakeGraceTime": 90,
     "shutdownGraceTime": 15,
     "shutdownCommand": "sshpass -p 'raspberry' ssh -oStrictHostKeyChecking=no pi@192.168.1.251 sudo shutdown -h now"
   },
   {
     "accessory": "NetworkDevice",
     "name": "My NAS",
     "host": "192.168.1.148",
     "log": false,
     "broadcastAddress": "172.16.1.255"
   }
]
```

For more configuration examples, please see the wiki.

### Required configuration

| Key       | Description                                       |
| --------- | ------------------------------------------------- |
| accessory | The type of accessory - has to be "NetworkDevice" |

### Optional configuration

#### Accessory information

| Key          | Description                                                                                                                                                                               |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name         | The name of the device - used in HomeKit apps as well as Siri, default `My NetworkDevice`                                                                                                 |
| manufacturer | The manufacturer of the accessory. Defaults to "homebridge-wol-ergin"                                                                                                                     |
| model        | The model name of the accessory. Defaults to "NetworkDevice"                                                                                                                              |
| serialNumber | A unique id for the accessory. Defaults to a random id which is reset each time the server restarts.                                                                                           |

#### Pinging

| Key           | Description                                                                                                |
| ------------- | ---------------------------------------------------------------------------------------------------------- |
| host          | The IP address or hostname to ping in order to receive current status                                      |
| pingInterval  | Ping interval in seconds, only used if `host` is set, default `2`. Same as that for pinging with a command |
| pingsToChange | The number of pings necessary to trigger a state change, only used if `host` is set, default `5`           |
| pingTimeout   |  Number of seconds to wait for pinging to finish, default `1`                                              |

#### Pinging using a command

| Key                | Description                                                                                                                                                                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| pingCommand        | Command to run in order to know if a host is up or not. If the command exits successfully (zero as the exit code) the host is considered up. If an error is thrown or the command exits with a non-zero exit code, the host is considered down |
| pingCommandTimeout | Timeout for the ping command in seconds. Use 0 (default) to disable the timeout                                                                                                                                                                |
| pingInterval       | Ping interval in seconds, only used if `host` is set, default `2`. Same as that for pinging                                                                                                                                                    |

_Note: these settings render those mentioned in the above section useless._

#### Turning on

| Key                 | Description                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| mac                 | The device's MAC address - used to send Magic Packets. Allows any format such as `XX:XX:XX:XX:XX:XX` or `XXXXXXXXXXXX`   |
| broadcastAddress    | The broadcast address to use when sending the Wake on LAN packet. Only used in rare cases. Defaults to `255.255.255.255` |
| startCommand        | Command to run in order to start the machine                                                                             |
| startCommandTimeout | Timeout for the start command in seconds. Use 0 (default) to disable the timeout                                         |
| wakeGraceTime       | Number of seconds to wait after startup before checking online status and issuing the `wakeCommand`, default `45`        |
| wakeCommand         | Command to run after initial startup, useful for macOS users in need of running `caffeinate`                             |
| wakeCommandTimeout  | Timeout for the wake command in seconds. Use 0 (default) to disable the timeout                                          |

#### Turning off

| Key                    | Description                                                                           |
| ---------------------- | ------------------------------------------------------------------------------------- |
| shutdownCommand        | Command to run in order to shut down the remote machine                               |
| shutdownGraceTime      | Number of seconds to wait after shutdown before checking offline status, default `15` |
| shutdownCommandTimeout | Timeout for the shutdown command in seconds. Use 0 (default) to disable the timeout   |

#### Logging

| Key      | Description                                                                                                         |
| -------- | ------------------------------------------------------------------------------------------------------------------- |
| logLevel |  The syslog log level to use such as `Debug`, `Info`, `Error`. The default is `Info`. Use `None` to disable logging |

#### Miscellaneous

| Key         | Description                                                                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| returnEarly | Whether or not to let the plugin return early to mitigate Siri issue. Defaults to `false` |

<a id="lifecycle"></a>

## Lifecycle

Whenever Homebridge starts, the plugin will check the state of all configured devices. This is done by pinging or executing the `pingCommand` once, depending on the configuration. If the pinging or `pingCommand` executes successfully, the device is considered online and vice versa.

The pinging by actual pings or use of the `pingCommand` will continue in the background, monitoring the state of the device. If `pingsToChange` (defaults to 5) pings have the same result and that result is not the current state, the state of the device will be considered changed. If `pingCommand` is configured, it is used instead of actual pings and will result in an immediate state change.

This pinging will result in state changes between online and offline.

Whenever you flick a switch to its on position via HomeKit, the device is marked as "turning on". Then a WoL packet is sent to the device if a MAC address is configured and if a `startCommand` is configured, it is also executed immediately. After the device has been started, it is marked as online. Then the plugin will wait for the time configured by `wakeGraceTime` (defaults to 45s). If a `wakeCommand` is configured, it will be called after the wait is over. Once all commands are completed, the device will be monitored again by the pinger - switching its state automatically.

Whenever you flick a switch to its off position via HomeKit, the device is immediately marked as turning off. A shutdown command is executed if it is configured. After the command's completion (or directly if none is configured), the plugin will wait for `shutdownGraceTime` (defaults to 15s) before continuing to monitor the device using the configured pinging method.

<a id="faq"></a>

## Notes and FAQ

### Permissions

This plugin requires extra permissions due to the use of pinging and magic packages. Start Homebridge using `sudo homebridge` or change capabilities accordingly (`setcap cap_net_raw=pe /path/to/bin/node`). Systemd users can add the following lines to the `[Service]` section of Homebridge's unit file (or create a drop-in if unit is packaged by your distro) to achieve this in a more secure way like so:

```
CapabilityBoundingSet=CAP_NET_RAW
AmbientCapabilities=CAP_NET_RAW
```

### Waking an Apple computer

The Macbook configuration example uses `caffeinate` in order to keep the computer alive after the initial wake-up.

### Controlling a Windows PC

The Windows configuration example requires the `samba-common` package to be installed on the server. If you're on Windows 10 and you're signing in with a Microsoft account, the command should use your local username instead of your Microsoft ID (e-mail). Also note that you may or may not need to run `net rpc` with `sudo`.

### SSH as wake or shutdown command

The Raspberry Pi example uses the `sshpass` package to sign in on the remote host. The `-oStrictHostKeyChecking=no` parameter permits any key that the host may present. **This usage is heavily discouraged. You should be using SSH keys to authenticate yourself.**

### Secrets in the configuration

Using username and passwords in a command is **heavily discouraged** as this stores them in the configuration file in plaintext. Use other authentication methods or environment variables instead.

### Development

```
# Clone project
git clone https://github.com/ozturkergin/homebridge-wol.git && cd homebridge-wol

# Install dependencies
npm install

# Build the project
npm run build

# Link it (if running homebridge locally)
npm link

# Make sure linting passes
npm run lint

# Try to start Homebridge with the Homebridge Config UI X locally - available on localhost:8080
# with default credentials admin:admin
npm run test

# Run Homebridge, Homebridge Config UI X and Homebridge WoL on the current maintenance node LTS version
docker-compose -f integration/docker-compose.yml up --force-recreate

# If you make changes to the code base, you may have to rebuild the containers before running the above command
docker-compose -f integration/docker-compose.yml build
```

The things to look out for when running the dockerized environment is:

1. Is the configuration UI working correctly? That is, is the `config.schema.json` up to date?
2. Can you interact with the accessories as expected? See the logs for full insights.

The accessory `MacBook Pro` is fully hooked up to simulate a real world MacBook, with proper shutdown over SSH as well as a custom ping command.

The `Generic` accessory features regular pinging using ICMP messages over IPv4.

The `Localhost` accessory represents the same instance as that running the Homebridge server. It features only mock configuration. It does feature the full range of configurations available, which should help debugging the configuration UI.
