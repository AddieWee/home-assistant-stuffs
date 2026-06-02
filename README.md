# Home Assistant Setup Documentation

Home Assistant Installation
---

### Install HAOS

Installation steps are for a **generic x86-64 PC**.

Follow the official guide:

> https://www.home-assistant.io/installation/generic-x86-64

### Network Setup

Choose one of the following options.

#### Option 1: Ethernet (Recommended)

1. Connect an Ethernet cable.
2. Boot Home Assistant.
3. Wait for Home Assistant to obtain an IP address.
4. Test connectivity:
   ```bash
   ping google.com
   ```

#### Option 2: Wi-Fi

In **HAOS CLI**, connect to WiFi

```bash
   > login # Enters prompt
   > nmcli radio # Checks if WiFi is working
   > nmcli device wifi rescan # Scans for WiFi networks
   > nmcli device wifi # Lists available WiFi networks
   > nmcli device wifi connect "YOUR_SSID" password "WIFI_PASSWORD" # Connects to network
```
   
Successful connection message
   
```bash
   Device 'wlan0' successfully activated with....
```

#### Test connectivity

```bash
ping google.com
```

## Home Assistant Apps

1. Login to **HAOS UI**

2. Settings > Apps > Install app 

### Studio Code Server

Search for `Studio Code Server`

1. Install

2. Start

3. Select optionals

   - Show in sidebar
### SSH

Search for `Terminal & SSH`

1. Install

2. Start

3. Select Optionals

   - Start on boot
   - Show in sidebar

## Remote Access & Security

### Remote Tunnel

> Reference: https://github.com/homeassistant-apps/app-cloudflared/wiki/How-tos#how-to-configure-remote-tunnel

#### CloudFlare - Configurations

##### Set up Routes in CloudFlare
1. Cloudflare > Zero Trust > Network > Connectors > Create a tunnel > Cloudflared

2. Provide a name

3. Note down the token. It starts with `ey.....`

4. Next

5. Give a subdomain

6. Service
   - Type: `http`
   - URL: `homeassistant:8123`

7. Save

#### HAOS - CloudFlared Set Up

1. Login to **HAOS UI**

2. Settings > Apps > Install app > 3 dots (top right) > Repositories > Add

3. Add Repository: `https://github.com/homeassistant-apps/repository`

4. Go back to: `Settings > Apps > Install app`

5. Search for `Cloudflared` (Refresh if not found)

6. Install

7. Optionals
   - Start on boot
   - Watchdog

8. Sidebar > Studio Code Server > `configuration.yaml` > Add these lines at the bottom
   ```yml
   http:
      use_x_forwarded_for: true
      trusted_proxies:
         - 172.30.33.0/24
   ```

9. Settings > System > power (top right) > Restart Home Assistant

### mTLS

#### Create Client Certs

1. Cloudflare > domain > SSL/TLS > Client Certificates > Create Certificate > Continue

2. Copy the **Certificate**

3. On your PC, open **Notepad** > Paste the **Certificate** > Save
   - Filename: `haos_mtls.p12`
   - Save as type: `All files`

4. Back to Cloudflare, copy the **Private Key**

5. On your PC, open **Notepad** > Paste the **Private Key** > Save
   - Filename: `haos_mtls.key`
   - Save as type: `All files`

6. On your PC, generate `p12` keystore. This can be done with WSL.
   
   1. Open WSL

   2. Navigate to location of certificate & private key (Assuming it's the Desktop)

      ```bash
      cd /mnt/c/Users/<your-windows-username>/Desktop
      ```

   3. Generate a `p12` keystore. 

      ```bash
      openssl pkcs12 -export -out haos_mtls.p12 -inkey haos_mtls.key -in haos_mtls.pem
      # A password is required when creating the keystore. Please remember it
      ```

#### Enforce mTLS

##### Whitelist Google Workspace IPs

1. Get list of **Google Workspace IPs**: `https://www.gstatic.com/ipranges/goog.json`

2. Create a custom list for google workspace IPs: `Cloudflare > Manage account > Configurations > Lists > Create list`
   - Identifier: `google_workspace_ip`
   - Description: `Source from: https://www.gstatic.com/ipranges/goog.json`

3. Upload the list from Gstatic as CSV.

4. Save

##### Create custom rule

1. Cloudflare > domain > Security > Security Rules > Create Rule > Custom rule

2. Under: `When incoming requests match…`

   | Field | Operator | Value |
   |-|-|-|
   | Client Certificate Verified | equals | false |
   | IP Source Address | is not in list | google_workspace_ip |
   | Hostname | is in | `your_domain` |

3. Under: `Then take action…`
   - Choose action: `Block`

4. Under: `Place at`
   - Select order: `Last`

5. Save

#### Install Client Certs

##### Windows

1. Double click the `haos_mtls.p12` file.

2. Follow through the steps. Input the required password which was provided in the step above when creating the `p12` file.


##### Google Chrome / Brave

> Steps may differ from each browser.

1. Settings > Privacy & security > Security > Manage certificates > Installed by you > Import

2. Select the `haos_mtls.pem` file.

###### IOS

1. Send the `haos_mtls.p12` file to your **iPhone**. Save it to **Files**

2. Open **Files**, tap on `haos_mtls.p12`

3. Choose a Device: `iPhone`

4. iPhone Settings > Profile Downloaded > Install > Enter Device Passcode > Provide keystore password

## Integrations

### Google Home

#### Link Google Assistant with Home Assistant

Follow the official guide:

> https://www.home-assistant.io/integrations/google_assistant/#manual-setup-if-you-dont-have-home-assistant-cloud


#### Set up automated Google Workspace IP updates

This is optional and only required if MTLS is enabled, and we want to whitelist Google's IPs from it.

##### Get Cloudflare Account ID & Create Token

1. Get the account ID: `Cloudflare > Account home > Top right menu > Copy account ID`

2. Create a token:
   
   1. Cloudflare > Top right menu > Profile > API Tokens

   2. Fill in:
      Token name: `HAOS_TOKEN`
      Permissions: `Account` | `Account Filter List` | `Edit`
   
   3. Continue to summary > Create Token


##### Install Pyscript

HAOS > Sidebar > HACS > Download

##### Configure Pyscript

1. HAOS > Sidebar > Studio Code

2. In `configuration.yaml`
   ```yaml
   pyscript:
      allow_all_imports: true
      hass_is_global: true

   logger:
      default: info
      logs:
         custom_components.pyscript: info
   ```

3. HAOS > Settings > Devices & Services > Add integration > Search for `pyscript` > Submit

4.  Settings > System > power (top right) > Restart Home Assistant 

##### Set Up Script

1. HAOS > Sidebar > Studio Code

2. Create a new directory
   ```bash
   config/pyscript
   ```

3. Create an import file: `pyscript/requirements.txt`
   ```bash
   # List of imports needed
   cloudflare
   requests
   ```

4. Create the script file: `pyscript/cf_update_google_ips.py`
   ```python
   from cloudflare import Cloudflare
   import requests

   ACCOUNT_ID = "YOUR_ACCOUNT_ID"
   TOKEN = "YOUR_API_TOKEN"
   LIST_NAME = "google_workspace_ip"

   # @pyscript_executor is used to make blocking calls
   @pyscript_executor
   def get_google_addresses():
      r = requests.get("https://www.gstatic.com/ipranges/goog.json")
      r.raise_for_status()
      ips = [ entry['ipv4Prefix'] for entry in r.json()['prefixes'] if 'ipv4Prefix' in entry ]
      ips = ips + [ entry['ipv6Prefix'] for entry in r.json()['prefixes'] if 'ipv6Prefix' in entry ]
      return ips

   @pyscript_executor    
   def update_addresses(ips):
      client = Cloudflare(api_token=TOKEN)
      
      # Get ID of the list name we specified
      page = client.rules.lists.list(account_id=ACCOUNT_ID)

      list_identifier = next(
         (item.id for item in page.result if item.name ==LIST_NAME),
         None
      )

      # Replace the entire list
      req_body = [
         {
               "ip": ip,
         }
         for ip in ips
      ]

      resp = client.rules.lists.items.update(
         list_id=list_identifier,
         account_id=ACCOUNT_ID,
         body=req_body,
      )

      return resp

   # Will trigger every 12AM
   @time_trigger("cron(0 0 * * *)")
   @service
   def update_google_ips():

      ips = get_google_addresses()
      log.info(f"List of Google Workspace IP Adresseses: {ips}")
      page = update_addresses(ips)
      log.info(f"{page}")
   ```

> When in doubt, restart HAOS

> References: 
> - https://developers.cloudflare.com/api/python/resources/rules/subresources/lists
> - https://hacs-pyscript.readthedocs.io/en/latest/tutorial.html

#### Cloudflare Security Rules

Enabling this would cause Google Assistant to not be able to connect to the HAOS instance as Cloudflare will treat Google's requests as a bot.

##### Disable Bot Fight mode

1. Cloudflare > Domains > Security > Settings > Bot fight mode

2. Toggle `off`

##### Local fulfillment

https://www.home-assistant.io/integrations/google_assistant/#enable-local-fulfillment

### LocalTuya & Tuyalocal

#### Install HACS

Follow the official guide:
> Reference: https://www.home-assistant.io/blog/2024/08/21/hacs-the-best-way-to-share-community-made-projects/#how-to-install

  3. https://www.hacs.xyz/docs/use/configuration/basic/#to-set-up-the-hacs-integration

#### Prerequisites

- HA must be on same WiFi network as Tuya devices.
- Tuya device must be linked to your Tuya app.

#### Set Up LocalTuya

> Reference: https://github.com/rospogrigio/localtuya

##### Creating a Tuya Developer Platform account

1. Create an account: https://us.platform.tuya.com

2. Create a new Project: `Cloud > Project Management > Create Cloud Project`
   - Industry: `Smart Home`
   - Development Mehtod: `Smart Home`
   - Data Center: `Western Americ Data Center`

3. Get the authorization QR code: `Open Project > Devices > Link App Account > Add App Account > Tuya App Account Authorization`

4. In your Tuya app, scan the authorization QR code: `Add (top right) > Scan`

##### Getting Tuya device info

1. Get device IDs: `Subtab  > All Devices`

2. Cloud > API Explorer > Device Management > Query Device Details in Bulk:
   1. Input the `device_id(s)` that you've gotten from above here separated by comma. Eg: abc,def
   2. Submit Request
   3. Take note of all the `local_key` of each device

3. Get the list of datapoints (DP) of each Tuya device: `Cloud > API Explorer > Device Control > Query Properties`
   1. Input a single `device_id`
   2. Submit Request
   3. Take note of the list of DPs
   4. Repeat for all your devices


4. Get the possible values of each DP of a specific Tuya device: `Cloud > API Explorer > Device Control (Standard Instruction Set) > Get the specifications and properties of the device`
   1. Input a single `device_id`
   2. Submit Request
   3. Take note of the list of `type` for each DP
   4. Repeat for all your devices

##### Setting up device in LocalTuya

1. Navigate: `Settings > Devices & Services > Integrations > LocalTuya > Add entry`
   1. Leave everything empty
   2. Check `Do not configure a Cloud API account`
   3. Submit

2. Under `Integration entries > Gear icon > Add a new device`
   1. Either pick a discovered device or `...` to add an undiscovered device.
   2. `Submit`

3. Configure entity
   1. Fill in:
      - Name: Eg: `Living Room Switch
      - Host: IP of device
      - Device ID: `From Getting Tuya device info, step 1`
      - Local Key: `From Getting Tuya device info, step 2`
   2. Submit
   3. Entity type selection, based on `Getting Tuya device info, step 4`:
      
      | Type (Tuya Dev Platform) | Entity Type (HA UI) |
      |-|-|
      | Boolean | switch |
      | Integer | number |
      | Enum | select |
      | String | String |

   4. In this form, the ID refers to the DP IDs `from Getting Tuya device info, step 4`
      - Friendly Name: Can input a user friendly name based on the `code` in `Getting Tuya device info, step 3`
      - Current: `Select based on DP ID`
      - Restore the last set value after a lost connection: `Select based on what you prefer`
      - Submit
   5. Uncheck `Do not add any more entities`
   6. Submit
   7. Repeat `Configure entity, step 4 to 6`

4. Repeat Setting up device in LocalTuya, step 3 for all your devices


### Tuyalocal

#### Set Up Tuyalocal

> Reference: https://github.com/make-all/tuya-local

#### Adding a custom device

> Reference `README` in: https://github.com/make-all/tuya-local/tree/main/custom_components/tuya_local/devices

1. Create a new YAML File in: `/config/custom_components/tuya_local/devices/` with any name, eg: `a_wall_switch_2_gang_sensor.yaml`

2. File content sample
   ```yaml
   name: Custom Wall Switch 2 Gang with Sensor
   products:
   - id: motal1vvze8wmzib # Device Management > Query Device details > 'product_id'
   entities: # Device Control > Query Things Data Model
   - entity: switch  # HA's data type
      name: switch_1 # services.properties[].code
      dps:
         - id: 1  # services.properties[].abilityId
            type: boolean
            name: switch # name is kind of mandatory
   - entity: switch
      name: switch_2
      dps:
         - id: 2
            type: boolean
            name: switch
   - entity: select
      name: relay_status
      category: config
      dps:
         - id: 14
            type: string
            name: option
            mapping:
               - dps_val: "off" # Double quotes for on/off are mandatory, else HA will parse it as a boolean
                  value: "off"
               - dps_val: "on"
                  value: "on"
               - dps_val: memory
                  value: memory
   - entity: switch
      name: backlight
      category: config
      dps:
         - id: 16
            type: boolean
            name: switch
   - entity: binary_sensor
      name: occupancy # does not follow Tuya API's field 'code'
      class: occupancy
      dps:
         - id: 102
            name: sensor # This is needed
            type: string
            readonly: true
            mapping: # Provided under `typeSpec`
               - dps_val: presence
                  value: true
               - dps_val: none
                  value: false
   - entity: number
      name: Presence Delay
      category: config
      dps:
         - id: 103
         name: value
         unit: s
         type: integer
         range:
            min: 5
            max: 86400
   - entity: number
      name: Sensitivity
      category: config
      dps:
         - id: 104
         name: value
         type: integer
         range:
            min: 1
            max: 25
   ```

#### Troubleshooting Tuyalocal 

Enable Debug logs for tuyalocal

1. `Tuyalocal > 3 dots (top right) > Enable Debug logging`

If Custom device YAML doesn't reflect:

1. Remove device

2. Restart Home Assistant

3. Add device again

If there's an error adding a device:

1. Validate the custom device's YAML

### Blocking Tuya Device's Internet Connection

1. Get IP address of Tuya devices

2. SSH to router

3. Run the following commands to drop traffic from the firewall:

   ```bash
   iptables -I FORWARD -s IP_OF_DEVICE -o ppp0 -j DROP
   iptables -I FORWARD -d IP_OF_DEVICE -i ppp0 -j DROP
   ```

4. Validate rules:

   ```bash
   iptables -L -v -n
   ```

## Troubleshooting

### Observability

#### Google Cloud Logs

Google Workspace > Select Project > Monitoring > Logs Explorer > Select date/time

#### Cloudflare Deny Logs

Cloudflare > Domains > melonlabs.uk > Security > Analytics > Events

> Can filter by path or RayId

#### Cloudflare Tunnel Access Logs

This does not provide historical info.

1. Cloudflare > Zero Trust > Network > Connectors > `tunnel_name` > Live logs

2. Begin log stream

#### LocalTuya Issues

`Connection to device succeeded but no datapoints found, please try again. Create a new issue and include debug logs if problem persists.`

Solution: In field `Manual DPS to add`, input the array of DPs

## Companion App

Download the app from PlayStore/App Store

### Temporarily Disable mTLS

1. Search for the mTLS rule: `Cloudflare > domain > Security > Security Rules`

2. At the right of the rule > 3 dots > `Disable`

### Set Up Companion App

1. Complete the setup process

2. You should be able to view your devices.

3. Add mTLS cert: `Top right > Settings > Servers > Select your server instance > Client Certificate`

4. Add the `p12` file here.

### Re-enable mTLS

1. Search for the mTLS rule: `Cloudflare > domain > Security > Security Rules`

2. At the right of the rule > 3 dots > `Enable`

> Validate by relaunching your companion app. It may request for your client certificate again, which then you should provide the `p12` file.

