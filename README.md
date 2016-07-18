# How to deploy dalite-ng on OpenStack server
 
This repository contains ansible deployment scripts allowing one to deploy dalite-ng to OpenStack server.


Create a OpenStack environment
------------------------------

1. Create a root volume that has 40GB size. It should be based on Ubuntu 14.04 image. 
   We suggest you use vanilla image from [ubuntu page](https://cloud-images.ubuntu.com/).
   First you need to upload images to your project, to do this follow 
   [documentation here](https://github.com/open-craft/doc/blob/master/howto-upload-images-to-openstack.md). 
   When images are uploaded you'll need to: `Volumes` -> `Create Volume` -> `Volume Source` choose `Image` -> `Select image`.
   
2. Create `vps-ssd-3` instance from this volume: (`Boot Source` -> `Volume` -> `Your root volume`). 

3. Create an additional blank volume for storing log and database dumps; we recommend 100GB. Attach it to the instance you created, and make a note of the device identifier it gets assigned (for example, `/dev/vdb` or `/dev/vdc`).

4. Make a note of the instance's SSH private key; either save it to your SSH config for that host's IP address, or save it in an id_rsa file with permissions set to 600.

5. SSH to the new instance using its private key; verify the public key fingerprint against the OpenStack log page before proceeding.

6. Create an Ansible `hosts` file containing (for example): 

        [dalite]
        dalite.harvardx.harvard.edu ansible_host=xxx.xxx.xxx.xxx
                
   Please note that while host name is largely irrelevant, this host must be in dalite group.   

Obtain database credentials
---------------------------

1. Create a `private-extra-vars.yml` file and store it somewhere safe, all configuration should be stored in that file. 
2. Obtain database credentials 
3. Save host to `MYSQL_DALITE_HOST`
4. Save database to: `MYSQL_DALITE_DATABASE`
5. Save credentials to: `MYSQL_DALITE_PASSWORD`, `MYSQL_DALITE_USER`
6. Save the device ID of the blank volume you attached for log and database dumps to `DALITE_LOG_DOWNLOAD_VOLUME_DEVICE_ID`.

   
Generate secrets
----------------

1. Generate tarsnap keys, these keys shouldn't end up on dalite-ng server, but instead store them somewhere safe. 
    1. Generate key for swift container backup 
    2. Generate key for dalite logrotate backup   
2. Generate tarsnap read write keys from the master key, see [tarsnap-keymngmt](http://www.tarsnap.com/man-tarsnap-keymgmt.1.html),    
3. Go to dead man's snitch at https://deadmanssnitch.com/. And generate three snitches, save them under:
    1. `DALITE_LOG_TARSNAP_SNITCH` --- this will monitor saving logs, 
       this snich should have daily interval
    2. `SANITY_CHECK_SNITCH`  --- this will monitor sanity check, 
       this snitch should have 15 minute interval        
    3. `BACKUP_SWIFT_SNITCH`  --- this will monitor backup of swift container
       this snitch should have hourly interval 
4. Generate various dailte secrets; 
   1. `DALITE_SECRET_KEY` --- a random string 
   2. `DALITE_LTI_CLIENT_SECRET` --- a random string
   3. `DALITE_PASSWORD_GENERATOR_NONCE` --- a random string
   4. `DALITE_LOG_DOWNLOAD_PASSWORD` --- a crypt compatible [encrypted password](http://linuxcommand.org/man_pages/mkpasswd1.html)
6. Obtain HTTPS certificate for dalite domain, and store it like that:

   ```
   
       DALITE_SSL_KEY: |
          -----BEGIN PRIVATE KEY-----
          data
          -----END PRIVATE KEY-----
    
       DALITE_SSL_CERT: |
          -----BEGIN CERTIFICATE-----
           data
          -----END CERTIFICATE-----
          -----BEGIN CERTIFICATE-----
           Parent certificate 
          -----END CERTIFICATE-----
    
   ```

5. Obtain OpenStack credentials to a project where media uploads will be stored, these should go to `DALITE_ENV`, 
   like this:
         
        DALITE_ENV:
          DJANGO_SETTINGS_MODULE: 'dalite.settings'
          OS_AUTH_URL: '...'
          OS_TENANT_ID: '...'
          OS_TENANT_NAME: '...'
          OS_USERNAME: '...'
          OS_PASSWORD: '...'
          OS_REGION_NAME: '...'

   and to: `BACKUP_SWIFT_RC` (yes you need to copy the same credentials in two ways, but ansible has no sensible 
   "copy variables" facility: 
   
    BACKUP_SWIFT_RC: |

      export OS_AUTH_URL='....'
      export OS_TENANT_ID='...'
      export OS_TENANT_NAME='...'
      export OS_USERNAME='...'
      export OS_PASSWORD='..'
      export OS_REGION_NAME='..'  
       
      
Last but not least encrypt the `private-extra-vars.yaml`:
 
   `ansible-vault encrypt host_vars/mysql.opencraft.com/private.yml --vault-password-file .vault-pass`      


Perform deployment
------------------

If the device identifier your external log volume was assigned is not /dev/vdc (the default we look for), then you'll need to pass it into the command.

    ansible-galaxy install -r requirements.yml -f
    ansible-playbook deploy-all.yml -u ubuntu --extra-vars private-extra-vars.yml

If you saved the instance's private SSH key to a separate file, rather than into your SSH configuration, you'll need to pass the `--private-key` argument to `ansible-playbook`, specifying the file where the private key can be found.
    
    
   