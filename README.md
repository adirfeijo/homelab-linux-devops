# Homelab : My Linux & DevOps Journey
My hands-on Linux and DevOps journey: repurposing laptops, self-hosting services, and documenting troubleshooting and automation.

## Why I Started
Over time, I realized that I had many online subscriptions, such as : Netflix (8€), Disney+ (11€), Amazon Prime Video (7€), Spotify (12€) and Google One (30€/ year mostly for my Google Photos).

The total amount I was paying each month was a lot : 40,5 €/month

And the annual amount : 486 €

The worst part, is knowing that the movie that I wanted to watch isn't available on the 3 streaming platforms that I pay for, or maybe it's not available anymore.
Try explaining this to some friends you invited home for a movie night.

What about this message : "Your storage is 91% full. Once you run out of space, you won't be able to send and receive emails, back up new photos... Upgrade your Google One plan to get more storage"

Thats when I thought : something needs to change!

I need to own my own movies (all of them), have all my music available for free for all my devices and have more control over my data.

And since I had some spare laptops, I knew that it would be possible.
That's where I got into the rabbit hole of homelabbing, searching on YouTube, reading articles...
Then I took action.

I'll be explaining everything in this repository and documenting my journey.

## Project Goals
The idea is to replace my current subscriptions with free open-source ones.

Google Photos --> Immich

Google One --> Nextcloud

Netflix, Disney+ and Prime Video and Spotify --> Jellyfin 

## Hardware
I started out with my wifes' old Sony Vaio Laptop from 2010. Then moved on to my other laptop : a friends old Lenovo L580.
Here are the specs.

Lenovo ThinkPad L580

Processor : Intel(R) Core(TM) i5-8350U CPU @ 1.70GHz

RAM : 8GB

Storage : 256GB

## Installation process

<img width="4080" height="3072" alt="l580 before" src="https://github.com/user-attachments/assets/5984810b-c06a-40e3-bf78-19d0ca2b75c5" />

As the image shows, I didn't have the easiest start. 

The laptop's screen is partially broken (upper left corner) and it has windows with an account and password I don't have access to.
The installation process wasn't easy.

1st step : 
Download ubuntu server iso.
Make your USB a bootable stick with the iso and Rufus.

2nd step : booting up with USB Stick 

1 - Turn off the laptop

2 - Connect the Lenovo L580 to an external monitor through HDMI

3 - Press power on button

4 - When the screen turns on, press F11 button several times. This enables to enter the BIOS to change the boot order.
By default, the boot order is Windows then the USB stick. We have to choose to boot with the USB stick thanks to the arrows and enter button

5 - Accept or refuse install options

Since my L580 computer wasn't connected to the router with e-thernet cable (which is recommended) I had to configure my laptop through wifi, which wasn't built-in. 

