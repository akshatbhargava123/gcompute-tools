# Git authentication tools for Google Compute Engine

The `git-cookie-authdaemon` uses the GCE metadata server to acquire an
OAuth2 access token and configures `git` to always present this OAuth2
token when connecting to googlesource.com or
[Google Cloud Source Repositories][CSR].

[CSR]: https://cloud.google.com/source-repositories/

## Setup

Launch the GCE VMs with the gerritcodereview scope requested, for example:

```
gcloud compute instances create \
  --scopes https://www.googleapis.com/auth/gerritcodereview \
  ...
```

To add a scope to an existing GCE instance see this
[gcloud beta feature](https://cloud.google.com/sdk/gcloud/reference/beta/compute/instances/set-scopes).

## Installation on Linux

Install the daemon within the VM image and start it running:

```
sudo apt-get install git
git clone https://gerrit.googlesource.com/gcompute-tools/
./gcompute-tools/git-cookie-authdaemon
```

The daemon launches itself into the background and continues
to keep the OAuth2 access token fresh.

### Launch at Linux boot

git-cookie-authdaemon can be started as a systemd service at boot.

```
# Write the service config
$ sudo cat > /etc/systemd/system/git-cookie-authdaemon.service << EOF
[Unit]
Description=git-cookie-authdaemon required to access git-on-borg from GCE

Wants=network.target
After=syslog.target network-online.target

[Service]
User=builder  # update to your user
Group=builder  # update to your group
Type=simple
ExecStart=/path/to/git-cookie-authdaemon  # update the path
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

# Reload the service configs
$ sudo systemctl daemon-reload

# Enable the service
$ sudo systemctl enable git-cookie-authdaemon

# Start the service
sudo systemctl start git-cookie-authdaemon

# Check the status of the service
systemctl status git-cookie-authdaemon
ps -ef | grep git-cookie-authdaemon

# Reboot and check status again.

```

## Installation on Windows

### Prerequisite

Install [Python 2.7](https://www.python.org/downloads/windows/) and
   [Git](https://git-scm.com/download) for Windows.

### Run interactively or in a build script

Run `git-cookie-authdaemon` in the same environment under the same user
git commands will be run, for example in either `Command Prompt`
or `Cygwin bash shell` under user `builder`. In Windows `Command Prompt`
`start` can be used to put the process into background.
```
python git-cookie-authdaemon --nofork
```

### Launch at Windows boot

It may be desired in automation to launch `git-cookie-authdaemon` at
Windows boot. It can be done as a scheduled task. The following is an
example on a Jenkins node:

1. The VM is created from GCE Windows Server 2019 or 2012R2 image.
1. It runs under `builder` account.
1. It is launched from a Bash shell. Cygwin is used here. Msys2 or Git
   Bash may work too but not tested.

How to create a scheduled task.

1. Launch `Task Scheduler` from an Administrator account.
1. Click `Create Task` in the right pane.
1. In `General` tab:
   1. Change user to the one running Jenkins node if it is different. You may
      want to run Jenkins node as a non-privileged user, `builder` in this
      example.
   1. Select `Run whether user is logged on or not`
1. In `Trigger` tab. Add a trigger
   1. Set `Begin the task` as `At startup`.
   1. Uncheck `Stop task if it runs longer than`.
   1. Check `Enabled`.
1. In `Actions` tab.  Add `Start a program`.
   1. Set `Program/script` as `C:\cygwin64\bin\bash.ext`,
   1. Set `Add arguments` as
      `--login -c /home/builder/git-cookie-authdaemon_wrapper.sh` (see note
      below)
1. Click `Ok` to save it.
1. Optional: click `Enable All Tasks History` in `Task Scheduler`'s right pane.
1. Add `builder` account to `Administrative Tools -> Local Security Policy ->
   Local Policies -> User Rights Assignment -> Log On As Batch Job`

Note: `/home/builder/git-cookie-authdaemon_wrapper.sh` is as below:

```
#!/bin/bash
exe=gcompute-tools/git-cookie-authdaemon
log=/cygdrive/c/build/git-cookie-autodaemon.log

# HOMEPATH and HOMEDRIVE are not set in a task scheduled at machine boot.
export HOMEPATH=${HOMEPATH:-'\Users\builder'}
export HOMEDRIVE=${HOMEDRIVE:-'C:'}

/cygdrive/c/Python27/python $exe --nofork >> $log 2>&1 # option "--debug" is also available.
```
