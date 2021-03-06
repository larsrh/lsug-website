# Table of Contents

1.  [The deployment pipeline](#org14a2e90)
2.  [Basic steps](#orgc15c5fc)
3.  [AWS resource modifications](#org3941465)
4.  [The User-Data script](#orgbf34438)
    1.  [Troubleshooting](#org3c17d3d)
5.  [The SSL Certificate](#orgdcd4503)


<a id="org14a2e90"></a>

# The deployment pipeline

By “deployment”, we mean taking the code in the master branch and
using it to run <http://www.lsug.co.uk>. The “pipeline” is the series of
steps involved in doing this.

If you’re a contributor, you won’t have the ability to view or change
this. These notes won’t be useful to you, but you might find them
interesting.


<a id="orgc15c5fc"></a>

# Basic steps

We release a new website every time a change is merged into the master
branch.

Usually, the only modifications are a new jar or assets that need to
be placed on the EC2 instance. The artifacts (the jar and assets) are
uploaded to S3 by `mill ci.synth`. The EC2 instance is configured to
download these and restart the website on reboot. We trigger a reboot
with `mill ci.reboot`.

The steps and EC2 instance configuration are completely coded in mill
tasks. They can be found using `mill resolve ci._`, with the code being
mostly in `web/scripts/*.sc`. These tasks are automated by a GitHub
action in `release.yml`.


<a id="org3941465"></a>

# AWS resource modifications

For more complex changes, such as changes to the AWS resources
(e.g. the EC2 instance), the deployment pipeline has some extra steps.
All AWS resources are managed by CloudFormation. A CloudFormation
update is triggered by `mill ci.deploy`.

If the User-Data script changes, CloudFormation puts the new script
onto the EC2 instance. *CloudFormation won’t terminate the
machine*. This means that the instance’s filesystem is preserved after
the deploy. *It won’t reboot the instance either* — if a reboot is
needed, it needs to be done with `mill ci.reboot`.


<a id="orgbf34438"></a>

# The User-Data script

The User-Data script governs what happens when an instance is first
created, as well as what happens on boot.

For example, it tells the instance to:

-   download java on creation
-   download the jar and assets on boot

If you change the boot instructions (the shell script at the end of
the User-Data string), these changes will be executed when the machine
next boots. This can be triggered with `mill ci.reboot`.

Changing the instance creation instructions is more nuanced — the new
instructions will be put onto the EC2 instance; but since it's already
been created, these changes will never be run. If you add any instance
creation instructions, you will need to run these manually by hopping
onto the host and running `cloud-init`.

For example, to run the `write_files` step:

    sudo cloud-init single --name write_files --frequency always

The meaning of each of these parameters is detailed in the [cloud-init
spec](https://readthedocs.org/projects/cloudinit/downloads/pdf/latest/).

Whether or not an instruction is run on instance creation or boot is
determined by the [cloud-init module](https://cloudinit.readthedocs.io/en/latest/topics/modules.html) describing it.


<a id="org3c17d3d"></a>

## Troubleshooting

cloud-init structures its configuration in a specific [directory
layout](https://cloudinit.readthedocs.io/en/latest/topics/dir_layout.html):

-   You can check the full User-Data at `/var/lib/cloud/instance/user-data.txt`.
-   The part run on boot can be found at `/var/lib/cloud/instance/scripts/user-data.txt`.

As always, you can find logs in `/var/log`:

-   `/var/log/cloud-init.log` will show which modules ran and which were skipped.
-   `/var/log/user-data.log` contains the output of the User-Data script ran on boot.


<a id="orgdcd4503"></a>

# The SSL Certificate

Arguably, the most complex part of the User-Data script is dedicated
to the SSL certificate.

The certificate is stored in S3 in a PKCS12 file. It is downloaded
during boot and its path and password are handed to the webserver as
arguments.

The certificate is renewed every few months by certbot. Certbot needs
a few files to be set up in the `/etc/letsencrypt/` directory. These are
downloaded from S3 on boot.

The `certbot-renew` systemctl timer is also enabled on boot. This is
somewhat redundant — if coded properly, it would be enabled once on
instance creation.

Note that this part of the pipeline could be simplified. Since these
are unchanging static files, they could be downloaded on instance
creation instead of on boot, so wouldn't need to be downloaded during
deployment at all.
