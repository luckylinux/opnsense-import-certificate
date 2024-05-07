# Introduction
Script to import a Letsencrypt SSL Certificate into a running OPNsense System.

# Basic Usage
```
Usage: php opnsense-import-certificate.php /path/to/certificate.crt /path/to/private/key.pem cert_hostname.domain.tld
```

# Automation example with dehydrated acme

```
./dehydrated -c -f config-lan
# push cert to opnsense
scp opnsense-import-certificate.php certs/yourhost.domain.tld/privkey.pem certs/yourhost.domain.tld/fullchain.pem root@youropnsensehost: \
&& ssh root@youropnsensehost 'php opnsense-import-certificate.php /root/fullchain.pem /root/privkey.pem yourhost.domain.tld'
```

# Setting up Scheduled Task in OPNSense
## References
  - Create Custom Actions in OPNSense: https://forum.opnsense.org/index.php?topic=2263.0
  - Some additional Notes: https://github.com/ftrojahn/opnsense-import-certificate/pull/1
  - Import Web GUI TLS Certificates automatically: https://github.com/ftrojahn/opnsense-import-certificate
  - Debugging Actions Issues: https://forum.opnsense.org/index.php?topic=21707.0


## Custom Actions
### Introduction
The Custom Actions are stored in the `/usr/local/opnsense/service/conf/actions.d/` Folder.

### File Names, Action Names, Tag Names and Content
#### Custom Actions (Generic)
Filename: `/usr/local/opnsense/service/conf/actions.d/actions_<action-name>.conf`

Contents:
```
[tag-name]
command: my-command
parameters: --my-arg1 value1 --my-arg2 value2 ...
type:script
message: Custom Message Describing what's going on
description: Custom Message Describing what's going on
```

**IMPORTANT**: without the `description:` line, this Custom Action will NOT be available in OPNsense Web GUI !

#### Custom Actions (Test)
Filename: `/usr/local/opnsense/service/conf/actions.d/actions_test.conf`

Contents:
```
[whoami]
command:whoami
parameters:
type:script_output
message:Who am I?
```

#### Custom Actions (Production)
Filename: `/usr/local/opnsense/service/conf/actions.d/actions_webgui_ssl_certificates.conf`

Contents:
```
[restart]
command: php "/usr/local/letsencrypt/opnsense-import-certificate.php" "/usr/local/letsencrypt/fullchain.pem" "/usr/local/letsencrypt/privkey.pem" "<MYHOSTNAME>.<MYDOMAIN>.<MYTLD>"
parameters:
type:script
message: Automatically import updated Web GUI Certificates
description: Automatically import updated Web GUI Certificates
```

**IMPORTANT**: I'm NOT sure that the TAG name must be "[restart]". I tried with "[reconfigure]" and I was still getting an Error.

## Create Custom Action
### Procedure (Generic)
1. Create Custom Action: `nano /usr/local/opnsense/service/conf/actions.d/actions_<action-name>.conf`
2. Update Actions List: `service configd restart`
3. Test that it works correctly from the Command Line: `configctl <action-name> <tag-name>`

### Procedure (Test)
1. Create Custom Action: `nano /usr/local/opnsense/service/conf/actions.d/actions_test.conf`
2. Update Actions List: `service configd restart`
3. Test that it works correctly from the Command Line: `configctl test whoami`
4. Returned value: `root`

### Procedure (Production)
1. Create Custom Action: `nano /usr/local/opnsense/service/conf/actions.d/actions_webgui_ssl_certificates.conf`
2. Update Actions List: `service configd restart`
3. Test that it works correctly from the Command Line: `configctl webgui_ssl_certificates restart`
4. Returned value: `OK`


# Adding a new Scheduled Task in OPNSense Web UI
Setup a CRON Job in OPNSense Web GUI.
1. Navigate to System -> Settings -> Cron
2. Click "+" to add a new Item
3. Hours/Minutes: Select the Time you wish the Scheduled Task to run (choose a time close-by to make sure that it's working the first time, e.g. 5 minutes from now)
4. Day of the month/Months/Days of the week: leave it set to ANY (*) so the Certificate will automatically be updated every Day (if a new Certificate was uploaded to the Router, that is)
5. Command: Select "Automatically import updated Web GUI Certificates"
6. Description: fill in a User-Friendly Description (I use "Automatically import updated Web GUI Certificates (Letsencrypt)")
7. Click: Apply
8. Click: Apply (AGAIN) !
9. Monitor the Logs in System -> Log Files. **Be sure to use the Multiselect and select ALL types of Messages**
10. Verify if the Custom Action returns `OK` (`Exit Code 0`, `returned OK`) or an ERROR (e.g. `Exit Code 1`, `... returned Error (1)` and `returned exit status 1`)

**NOTE**: I'm NOT sure if it's really needed to click "Apply" twice. I prefer to do so, in case the first time didn't really trigger.

# Refreshing a Scheduled Task in OPNSense Web UI if the Custom Action File was changed
**IMPORTANT**: if you changed the Custom Action in **ANY** way:
- Filename was changed (e.g. `/usr/local/opnsense/service/conf/actions.d/actions_name1.conf` was moved/renamed to `/usr/local/opnsense/service/conf/actions.d/actions_name2.conf`)
- Tag Name was changed (`tag1` -> `tag2`)
- Command to be Executed was changed
- Arguments were changed
- Description was changed
- Message was changed

Then you will **MOST LIKELY NEED TO PERFORM THESE ADDITIONAL PROCEDURES**.

I also found that the CRON Job setup in the Web GUI tends to fail with `Exit Code 1` or `... returned Error (1)` or `returned exit status 1` even though it was working correctly from the CLI.

This might be due to the fact that the Filename and Action/Tag are set when you "click" on the Action Description in the Web GUI.

But these are not/might not be refreshed if you modified e.g. the Tag or Filename, **even if** you issued the `service configd restart` Command correctly.

If you use the "Inspect Element" Feature on Chromium / Firefox Web Browser, you will see that the Action Name **and** the Tag Name are **BOTH** included in the Scheduled Task Setting (value="[action-name] [tag-name]" in the HTML) when you click on an Item in the Command Select Box:
```
<option value="webgui_ssl_certificates restart" selected="selected">Automatically import updated Web GUI Certificates</option>
```

In order to workaround this issue some additionnal steps (TEMPORARILY change the Command to be executed) might be/are required:
1. Update Actions List: `service configd restart`
2. Test that it works correctly from the Command Line: `configctl webgui_ssl_certificates restart`
3. Edit CRON Job in OPNSense Web GUI with a TEMPORARY FIX
4. Set TEMPORARILY the command to one of the Other OPNSense Commands (e.g. `Automatic Firmware Update`)
5. Click Save
6. Click Apply
7. Click Apply (AGAIN) !
8. Edit the CRON Job in OPNSense Web GUI for the REAL TIME now
9. Reselect the Appropriate Command "Automatically import updated Web GUI Certificates"
10. Click Save
11. Click Apply
12. Click Apply (AGAIN) !
13. Monitor the Logs in System -> Log Files. **Be sure to use the Multiselect and select ALL types of Messages**
14. Verify if the Custom Action returns `OK` (`Exit Code 0`, `returned OK`) or an ERROR (e.g. `Exit Code 1`, `... returned Error (1)` and `returned exit status 1`)

**NOTE**: I'm NOT sure if it's really needed to click "Apply" twice. I prefer to do so, in case the first time didn't really trigger.

# Notes
- Update Actions List: `service configd restart`
- Debug Issues: `configctl <action-name> <tag-name>`
- Debugging Issues for the <test> action name with tag <whoami>: `configctl test whoami`
- Debugging Issues for the <webgui_ssl_certificates> action name with tag <restart>: configctl webgui_ssl_certificates restart
