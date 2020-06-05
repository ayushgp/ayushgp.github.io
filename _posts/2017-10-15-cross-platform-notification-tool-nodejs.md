---
layout: post
title: Building a Cross Platform Desktop Notification Tool Using Node.js
---

Notifications are graphical elements that notify users about events without forcing them to react to it immediately.

## Why use desktop notifications?

Notifications have many use cases. For example:

1. A chat application uses them to notify users about new messages and mentions on group messages, etc.

2. Stock market applications use them to notify users about the price of a stock they bought dropping or the price crossing a certain user set threshold, etc.

3. News applications use them to notify users about breaking news and update them of the latest events in their field of interest, etc.

4. Notifies users about software updates and security risks to their PCs, etc.

System applications, third party applications, and, most recently, web applications, have started using desktop notifications to make their applications more interactive and engaging.

## Simple Notifications

You can create a simple notification using just an icon and some text very easily. For example, if you have an icon called img.jpg, you can just call the node-notifier API with the text and icon:

```
const notifier = require('node-notifier')

notifier.notify({
  'title': 'My notification',
  'message': 'Hello, there!'
});
```

Run this code using:

```
$ node index.js
```

You should see a notification on your OS that looks like:
![simple-without-icon.jpg](https://cdn.filestackcontent.com/MTg1hebTwSnBOuCJDlFm)

You can even use an icon with your notification by passing the icon's path to the object. For example:

```
const notifier = require('node-notifier')

notifier.notify({
  'title': 'My notification',
  'message': 'Notification with an Icon!',
  'icon': 'icon.png'
});
```

When you run this code, you should see your icon on the left of your notification:
![notification-icon.jpg](https://cdn.filestackcontent.com/haBXuEl7RGSFlE950MnL)

The following are defaults for properties that work seamlessly across platforms:

```
notifier.notify({
  title: 'My awesome title',
  message: 'Hello from node, Mr. User!',
  icon: path.join(__dirname, 'coulson.jpg'), // Absolute path (doesn't work on balloons)
  sound: true, // Only Notification Center or Windows Toasters
  wait: true // Wait with callback, until user action is taken against notification
}, function (err, response) {
  // Response is response from notification
});
```

## Event Handling

node-notifier also supports 2 events, `click` and `timeout`, that can be handled using the `notifier` object. For example, you can log that the user clicked on the notification. You also have the notifier object that you pass to create the notification. You can use it to identify the notification and take action accordingly.

```
notifier.on('click', function(notifierObj, options) {
  // Triggers if `wait: true` and user clicks notification
  console.log('The user clicked on the Notification!');
});
```

You can also use the `timeout` event handler in a similar way. It triggers if `wait: true` and notification closes.

## Complex Notifications

You can create very complex notifications using the node-notifier. But different systems have their own APIs for creating heavily customized notifications. node-notifier supports [Notification Center](https://github.com/mikaelbr/node-notifier#usage-notificationcenter) for macOS, and [Toaster](https://github.com/mikaelbr/node-notifier#usage-windowstoaster) and [Balloon](https://github.com/mikaelbr/node-notifier#usage-windowsballoon) (for earlier versions of Windows) for Windows. You can check out the options you have available for these in the links.

## Wrapping up

Whether you are building a CLI and want to integrate notifications in it, or have a full blown electron desktop app, you can use `node-notifier` to spawn interactive notifications on your users' machines, regardless of their OS.
