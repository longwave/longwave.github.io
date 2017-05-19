---
layout: post
title:  Debugging provisioning failures in Packer
---

Hashicorp's [Packer](https://www.packer.io/) is wonderful for baking AMI images on AWS, and it's amazing when you set up a build with an Ansible playbook and it works perfectly the first time. But when it fails somewhere during the build, and the error message is not enough to tell you what the actual problem is, it's not obvious how to debug it further.

`packer build` failed during a playbook run that included [Jeff Geerling's nginx role](https://github.com/geerlingguy/ansible-role-nginx) with the following output:

```
    amazon-ebs: RUNNING HANDLER [geerlingguy.nginx : restart nginx] ****************************
    amazon-ebs: fatal: [127.0.0.1]: FAILED! => {"changed": false, "failed": true, "msg": "Unable to restart service nginx: Job for nginx.service failed because the control process exited with error code. See \"systemctl status nginx.service\" and \"journalctl -xe\" for details.\n"}
```

Unfortunately when Packer fails it immediately cleans up the mess, giving you no access to the instance to run the suggested commands. Reading the documentation suggested adding the `-debug` switch which pauses before each step, but this didn't work well enough as I wasn't sure when it was about to fail, and in any case it failed and then cleaned up the instance without

Reading the documentation again I discovered the relatively newly added `-on-error=ask` switch which would stop immediately after the failure and ask me what to do. This prompt would be a good point to switch to another terminal and SSH to the instance, but that has its own problems: where is the instance, and how do I connect to it? By default Packer manages all the AWS infrastructure by itself, including creating the SSH keys used to connect to the instance. Reading the documentation again I discovered that you can override this and specific your own keys. So after uploading my public key to the Key Pairs section of the EC2 console, I could add the following to my Packer JSON config file:

```
    "ssh_keypair_name": "Dave",
    "ssh_agent_auth": "true",
```

This would build the image with the public key "Dave" stored on AWS, and use my local SSH agent to connect. Running `packer build -on-error=ask` now ended like this:

```
    amazon-ebs: RUNNING HANDLER [geerlingguy.nginx : restart nginx] ****************************
    amazon-ebs: fatal: [127.0.0.1]: FAILED! => {"changed": false, "failed": true, "msg": "Unable to restart service nginx: Job for nginx.service failed because the control process exited with error code. See \"systemctl status nginx.service\" and \"journalctl -xe\" for details.\n"}
...
==> amazon-ebs: Step "StepProvision" failed
==> amazon-ebs: [c] Clean up and exit, [a] abort without cleanup, or [r] retry step (build may fail even if retry succeeds)? 
```

The final step was to discover how to connect to the instance. Early in the Packer log is this output:

```
==> amazon-ebs: Waiting for instance (i-0735154eb19cd2078) to become ready...
```

This ID is enough to discover the IP address of the running build instance, by either looking it up in the EC2 console or using the command line:

```console
$ aws ec2 describe-instances --instance-ids i-0735154eb19cd2078 | jq '.Reservations[].Instances[].PublicIpAddress'
"52.57.56.214"
```

So now I could SSH to the instance to try and discover the issue. I need to connect as the same user specified in the Packer JSON config; in this case I was building a CentOS image so the username is "centos":

```console
$ ssh centos@52.57.56.214
The authenticity of host '52.57.56.214 (52.57.56.214)' can't be established.
ECDSA key fingerprint is SHA256:26UCZ7D5Bd6My3o25uTmdhjceJPojvJ0+GJxvRXum6A.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '52.57.56.214' (ECDSA) to the list of known hosts.

[centos@ip-172-31-30-231 ~]$ systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2017-05-18 14:05:03 UTC; 13min ago
     Docs: http://nginx.org/en/docs/
  Process: 11903 ExecStop=/bin/kill -s QUIT $MAINPID (code=exited, status=0/SUCCESS)
  Process: 11905 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf (code=exited, status=1/FAILURE)
 Main PID: 9883 (code=exited, status=0/SUCCESS)

May 18 14:05:02 ip-172-31-30-231 systemd[1]: Starting nginx - high performance web server...
May 18 14:05:03 ip-172-31-30-231 nginx[11905]: nginx: [emerg] invalid number of arguments in "fastcgi_intercept_errors" directive in /etc/nginx/conf.d/vhosts.conf:51
May 18 14:05:03 ip-172-31-30-231 nginx[11905]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 18 14:05:03 ip-172-31-30-231 systemd[1]: nginx.service: control process exited, code=exited status=1
May 18 14:05:03 ip-172-31-30-231 systemd[1]: Failed to start nginx - high performance web server.
May 18 14:05:03 ip-172-31-30-231 systemd[1]: Unit nginx.service entered failed state.
May 18 14:05:03 ip-172-31-30-231 systemd[1]: nginx.service failed.
```

Finally, the culprit turns out to be a badly formed nginx config file - a single missing semicolon that I didn't spot when writing the template file.

