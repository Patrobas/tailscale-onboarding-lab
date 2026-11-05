# Troubleshooting: the server booted but never joined the tailnet

## What I saw
The EC2 instance started normally, but it never appeared in the Tailscale Machines list. I could not reach it over Tailscale.

## What I checked
The auth key in the Tailscale admin console still showed as unused. That told me the server had never even tried to log in, so the problem was in the join command itself, not in the network or the firewall.

## Problem 1: a typo in one flag
The hostname flag was written as `--hosyname` instead of `--hostname`. Because of the typo the whole command was rejected, so nothing after it ran.

## Problem 2: a stray space in the auth key
After fixing the flag, the server still did not join. In AWS CloudShell I could see the line had been broken in the middle, which put a stray space inside the auth key. A key with a space in it is not a valid key, so the login failed.

## How I found both
I read the script line by line instead of guessing. Reading it slowly is what showed both mistakes.

## What the boot log showed
The server's boot log (EC2 console output) recorded the failure. The join command ended with this error, auth key hidden:

    /bin/bash: curl -fsSL https://tailscale.com/install.sh | sh tailscale up --authkey=<KEY> --hosyname=acme-tools --ssh: No such file or directory

Bash treated the whole line as a file name instead of running it, so Tailscale never started and the auth key was never used. The line also shows the `--hosyname` typo and a space inside the key.

## The fix
I corrected the flag to `--hostname` and removed the space from the auth key. Auth keys are single use and the first one was wasted, so I made a new one. The startup script only runs the first time a server boots, so instead of editing the old server I terminated it and launched a replacement with the fixed script.
## How I confirmed it worked
The server appeared in the Machines list as acme-tools, and I connected to it with SSH over Tailscale. SSH to the public IP still timed out, as it should.

## What I would do next time
- Check the auth key state in the admin console first. It shows quickly whether the server ever tried to log in.
- Keep the join command on one line, and check it with `tailscale up --help` before putting it in a script.
- Read the cloud-init log on the server straight away instead of guessing.
