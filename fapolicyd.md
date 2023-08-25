
# Summary

You are trying to use Mattermost and have `fapolicyd` enabled and want them to play nicely.

## Guide

1. Create a rule file
  The naming convention for your rule is **really** important here. It must come before the rule that's denying Mattermost. If you're using a stock `fapolicyd` then `80` works fine. You can check the deny rule with the troubleshooting steps.

  ```bash
  sudo touch /etc/fapolicyd/rules.d/80-mattermost.rules
  ```
2. Add the permissions into to the rule file.

  ```bash
  sudo vi /etc/fapolicyd/rules.d/80-mattermost.rules
  ```
  
  Rule File:
  ```
  allow perm=execute exe=/user/bin/sudo trust=1 : dir=/opt/mattermost/ all trust=0
  ## this one is required for plugins to load properly.
  allow perm=execute exe=/opt/mattermost/bin/mattermost : dir=/opt/mattermost all trust=0
  allow perm=execute exe=/user/lib/systemd/systemd trust=1 : dir=/opt/mattermost/ all trust=0
  ```
3. Check the rules will be applied
  
  This command should say `Ruels have changed and should be updated`
  ```
  sudo fagenrules --check
  /usr/sbin/fagenrules: Rules have changed and should be updated
  ```
4. Update the rules

  ```
  sudo fagenrules --load
  ```
5. Now restart mattermost.

## Troubleshooting

## Summary

If Mattermost or the plugin binaries cannot start, you can troubleshoot this with the below steps.

Additional troubleshooting steps can be found on the RHEL docs [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/assembly_blocking-and-allowing-applications-using-fapolicyd_security-hardening#ref_troubleshooting-problems-related-to-fapolicyd_assembly_blocking-and-allowing-applications-using-fapolicyd)

## Troubleshooting

1. Stop `fapolicyd`
    ```
    sudo systemctl stop fapolicyd
    ```
2. Test your issue with Mattermost right now. Was it fixed? Then continue onward.
3. Run the debug command
    ```
    sudo fapolicyd --debug
    ```
4. Look for any denies. You can store the above in a file, but it kept yelling at me so i didn't.
    ```
    rule=15 dec=deny_audit perm=execute auid=-1 pid=19735 exe=/opt/mattermost/bin/mattermost : path=/opt/mattermost/plugins/focalboard/server/dist/plugin-linux-amd64 ftype=application/x-executable trust=0
    ```

