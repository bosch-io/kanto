---
title: "Create and update the containers `Desired State`"
type: docs
description: >
    Create and update containers via `Desired State` using the `update-manage`
weight: 3
---

By following the steps below you will publish a simple `Desired State` specification via a publicly available Eclipse Hono sandbox and then the specification will be handled by the Eclipse Kanto Update Manager, which will trigger an OTA update on
the edge device.

A simple monitoring application will track the progress and the status of the update process.
### Before you begin

To ensure that all steps in this guide can be executed, you need:

* Debian-based linux distribution and the `apt` command line tool

* If you don't have an installed and running Eclipse Kanto, follow {{% relrefn "install" %}} Install Eclipse Kanto {{% /relrefn %}}

* If you don't have a connected Eclipse Kanto to Eclipse Hono sandbox,
  follow {{% relrefn "hono" %}} Explore via Eclipse Hono {{% /relrefn %}}

* The {{% refn "https://github.com/eclipse-kanto/kanto/blob/main/quickstart/hono_commands_um.py" %}}
  update manager application {{% /refn %}}

  Navigate to the `quickstart` folder where the resources from the {{% relrefn "hono" %}} Explore via Eclipse Hono {{% /relrefn %}}
  guide are located and execute the following script:

  ```shell
  wget https://github.com/eclipse-kanto/kanto/raw/main/quickstart/hono_commands_um.py

* Enable the `containers update agent` service of the `Container Management` by adding the ` "update_agent": {"enable": true}` property to the `container-management` service configuration (by default located at `/etc/container-management/config.json`)
  and restart the service: 
  ```shell
  systemctl restart container-management
  ```

### Publish the `Desired State` specification

First, start the monitoring application that requires the configured Eclipse Hono tenant (`-t`) and an optional filter parameter (`-f`). It will print all
received feedback events triggered by the device:

```shell
python3 hono_events.py -t demo -f demo/device/things/live/messages/feedback
```

The starting point of the OTA update process is to publish the example `Desired State` specification:
```shell
python3 hono_commands_um.py -t demo -d demo:device -o apply
```

The `Desired State` specification in this case consists of single domain section definition for the containers domain and a three container components - `influxdb`, `hello-world` and `alpine` image.

### Apply `Desired State` specification

The Update Manager receives the published `Desired State` to the local Mosquitto broker, splits the specification (in this case into single domain) and then
distributes the processed specification to the domain agents which initiates the actual update process logic on the domain agents side.

The update process is organized into multiple phases, which are triggered by sending specific `Desired State` commands (`DOWNLOAD/UPDATE/ACTIVATE`).

In the example scenario, the three images for the three container components will be pulled (if not available in the cache locally), created as containers during the `UPDATING` phase and
started in the `ACTIVATING` phase.

### Monitor OTA update progress

During the OTA update, the progress can be tracked in the monitoring application fot the `Desired State` feedback messages, started in the prerequisite section above.

The Update Manager reports at a time interval of a second the status of the active update process. For example:

```json
{
   "activityId":"e5c858cc-2057-41b0-bd5f-83aee0aad22e",
   "timestamp":1693201088401,
   "desiredStateFeedback":{
      "status":"RUNNING",
      "actions":[
         {
            "component":{
               "id":"containers:alpine",
               "version":"latest"
            },
            "status":"UPDATE_SUCCESS",
            "message":"New container instance is started."
         },
         {
            "component":{
               "id":"containers:hello-world",
               "version":"latest"
            },
            "status":"UPDATE_SUCCESS",
            "message":"New container instance is started."
         },
         {
            "component":{
               "id":"containers:influxdb",
               "version":"2.7.1"
            },
            "status":"UPDATING",
            "message":"New container created."
         }
      ]
   }
}
```

### List containers

After the update process is completed, list the installed containers by executing the command `kanto-cm list` to verify if the `Desired State` is applied correctly.

The output of the command should display the info about the three containers, described in the `Desired State` specification. The `influxdb` is expected to be in `RUNNING` state and
the other containers in status `EXITED`. For example :

```text
ID                                    |Name         |Image                               |Status   |Finished At |Exit Code
|-------------------------------------|-------------|------------------------------------|----------------------|---------
7fe6b689-eb76-476d-a730-c2f422d6e8ea  |influxdb     |docker.io/library/influxdb:1.8.4    |Running  |            |0
c36523d7-8d17-4255-ae0c-37f11003f658  |hello-world  |docker.io/library/hello-world:latest|Exited   |            |0
9b99978b-2593-4736-bb52-7a07be4a7ed1  |alpine       |docker.io/library/alpine:latest     |Exited   |            |0
```

### Update `Desired State` specification

To update the existing `Desired State` run the command below. The update changes affect two containers - `alpine` and `influxdb`. Being not present in the updated `Desired State` specification, the `alpine` container will be removed from the system. The `influxdb` will be updated to version 1.8.5. The last container - `hello-world` is not affected and any events will be not reported from the container update agent for this particular container.

```shell
python3 hono_commands_um.py -t demo -d demo:device -o update
```

### List updated containers

After the update process of the existing `Desired State` is completed, list again the available containers to the verify the `Desired State` is updated correctly.

The output of the command should display the info about the two containers, described in the `Desired State` specification. The `influxdb` is expected to be updated with the version 1.8.5 and in `RUNNING` state and `hello-world` container to be status `EXITED` with version unchanged. The `alpine` container must be removed and not displayed.

```text
ID                                    |Name         |Image                               |Status   |Finished At |Exit Code
|-------------------------------------|-------------|------------------------------------|----------------------|---------
7fe6b689-eb76-476d-a730-c2f422d6e8ea  |influxdb     |docker.io/library/influxdb:1.8.5    |Running  |            |0
c36523d7-8d17-4255-ae0c-37f11003f658  |hello-world  |docker.io/library/hello-world:latest|Exited   |            |0
```

### Remove all containers

To remove all containers, publish an empty `Desired State` specification (with empty `components` section):
```shell
python3 hono_commands_um.py -t demo -d demo:device -o clean
```

As a final step, execute the command `kanto-cm list` to verify that the containers are actually removed from the Kanto container management.
The expected output is `No found containers.`.