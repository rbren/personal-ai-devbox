# Things you have to do by hand

Agents can't do everything for you. Twilio in particular seems hesitant to
give out phone numbers to just any robot scraping their website.

## Provision hardware.

First, you need somewhere to run this thing.

The most obvious answer here is your laptop. I don't recommend it. It's hard to be in flow state
when you're worried the agent might delete all your photos. Better to use a sandbox.

You could use Docker on your laptop. Not a terrible solution. Maybe mount directories you want the agent to work with.

You could buy a separate machine, just for this. Also not a bad solution, but expensive and inconvenient.

I recommend renting a VM in the cloud (I use DigitalOcean). Easy to try it out for a few hours or months. Plus you can
set it up so your workstation is available anywhere in the world.

## Take security seriously.

These interfaces can do literally anything on the host machine. You're responsible for your own security.

And that's before we weave in AI. Today's agents often lack what humans think of as "common sense", and are easily conned.
You're responsible for what your agent does.

You should almost definitely put this machine behind a firewall. If you're renting from one of the major clouds, you can
create an IP allowlist for your home IP. Do you add your local cafe too? I dunno friend, your call.

You should also almost definitely put basic authorization in place, with a strong password.
The agent knows how to set this part up for you.

Be careful what secrets and MCP servers you give to the agent. Even with incoming connections blocked, your agent could
browse the wrong webpage and get prompt injected. Stay safe. The OpenHands agents we're using have support
for security analyzers and hooks--take advantage of them.

## Set up integrations.

You can add secrets and MCP servers to give your agent access to all sorts of things. You'll need to generate
the relevant API keys yourself.

Set up the SMS app by adding some Twilio credentials to your secrets. Make sure the SMS app only accepts SMS messages from your own number.
See: [Take security seriously](#Take-security-seriously)
