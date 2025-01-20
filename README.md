# Home-Assistant-Send-Notification
Home Assistant script template for sending a notification to Home Assistant mobile apps or HTML5 endpoints

## Overview
I use notifications all of the time, and they’re very powerful. But they can also be very hard to use. Every time I create a new notification I have to look up the YAML for specifying a tag or group, and more advanced features like critical notifications, badging, or including an image are even more complex. Features vary by platform, and you often have to carefully read through the developer docs 1 multiple times to understand all of the capabilities.

To make this easier, I created a script with as many of the notification features as possible. You can send and update notifications, remove notifications, set badge counts, and much more. The notifications also support actions, attachments, and even triggering Siri Shortcuts. The script has quite a few fields (I wish I could dynamically populate the script based on your selection), but I still think it’s far easier to use since you only have to set cross-platform features once, even if the syntax is different per-platform.

Note: I primarily use iOS, so I haven’t yet added some of the Android-only features like notification LED colors, alert once, etc. But it includes most other features so far. It also supports both mobile (iOS/Android) and HTML5 notifications.

## Pre-requisites / setup instructions:

1. You must already have mobile 2 and HTML5 notifications 6 set up
2. Copy the code for the script ([send_notification.yaml](https://github.com/MasterDevwi/Home-Assistant-Send-Notification/blob/main/send_notification.yaml)) to a new script in Home Assistant. Note: You’ll need to edit two things before it’ll work properly.
3. In the send_to field, open the Selector Options and modify the groups and users to match your setup. I’d recommend creating notification groups in notify.yaml to make things easier (for example, my notify.notify_family includes notify.notify_william and notify.notify_amy). The values should match the names of the notify services for each group/user/device (minus the “notify.” prefix).
4. The first action in the sequence is called Map browsers. It maps the same notify entities above to the HTML5 devices you want to send notifications to for that user/group. You’ll need to update this list to match your send_to options above and the device names you specified when enabling HTML5 notifications on each device.
5. The script should now be working!

I hope this helps a few folks! It’s my first draft, so please let me know if you have any feedback.
