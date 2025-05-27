# Useful Homeassistant Things
A repository for my useful homeassistant guides, tips and blueprints.

## [Optimising Home Assistant's Database](/database/README.md)

An absolute must and probably one of my most important reads for anyone using Home Assistant. This will prevent most database errors that arise in Home Assistant.

## [Home Assistant's Database Blueprints](/homeassistant-purge-entities-automation/README.md)

These help clean up those pesky entities which are using up all of your database space.

## Home Assistant OS Virtualisation

Home Assistant is best run as a full OS (i.e. Home Assistant OS). While installing Home Assistant as a container (e.g. docker) is supported, it is not recommended and you will lose functionality. Home Assistant Supervised and Home Assistant Core installations are no longer supported. If you are anything like me, you installed Home Assistant on the nearest hardware you had available, this started with a Raspberry Pi 3 and USB SSD for me, onto an old small form factor PC and now on a dedicated small form factor PC I bought for this purpose.

When I installed on the small form factor PCs (and given it consumes 10x the power of a Raspberry Pi) I wanted my install to do more (media server, NAS, adblocking dns, etc.). This functionality *can* come built-in to Home Assistant through the addon system but by doing so:

1. You must use the addons supported by Home Assistant.
2. Customisation of those addons can be very cumbersome (e.g. I want to upload files bigger than 50MB to my own wiki).

You regularly hear that virtualisation overhead should be around 10-20% and so you might be encouraged to install your favourite flavour of linux and use VirtualBox to run Home Assistant OS (it is, after all, the first recommended way to install Home Assistant OS on Linux (at time of writing)). What you should be aware of is that VirtualBox's overhead when running Home Assistant is significant - my average CPU utilisation when using VirtualBox is around 30% compared to using a dedicated hypervisor where it's around 5%. My results aren't unique, I just didn't go fully down the rabbit hole when doing my first install (see: https://shatteredsilicon.net/virtual-performance-or-lack-thereof/).

I now use a proper virtualisation environment (Proxmox) and haven't looked back since.
