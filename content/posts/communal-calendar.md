---
title: Communal Calendar
description: A Google calendar with edit access for everyone who needs it, without the Human Sacrifice.
date: 2025-06-22
authors:
  - name: Andy Casey
    link: https://astrowizici.st
    image: ../../me.png

  - name: Zach Way
    link: https://astro.gsu.edu/~way/
    image: ../zach-way-e1632157542151.jpeg
tags:
  - career-advice
  - mentoring  
---

I'm part of the [Sloan Digital Sky Survey](https://www.sdss.org) (SDSS). There are a lot of people in SDSS, and there are a lot of telecons. All the telecon times were listed in different areas of the Wiki, in different timezones, and those telecons would routinely reschedule each semester to accommodate changing constraints of the participants. At any given time it was pretty hard to collate all the currently scheduled telecons, let alone convert them to your current timezone.

One sensible graduate student, [Zach Way](https://astro.gsu.edu/~way/), suggested that we should create a centralised calendar for all the telecons, so that people could see at a glance when they were happening, and avoid scheduling conflicts.

It's a great idea. But it's way harder than it sounds!

Unfortunately, making that calendar editable by lots of people (but not _everyone_) is a pain in the ass. If you trusted all humans then you could just make the calendar editable to _everyone_. But if you don't want to do that, it gets hard. Everyone is at different institutions, so you can't just say "_Everyone at my institution_". If you want to give edit access to lots of people at different institutions then you need to add each person by their email address. And some people have multiple email addresses, and they don't want to have to be logged in to _Account X_ at any given time.

We want trusted people to be able to give themselves access through whatever accounts they have. And we want to do that without human involvement.

Let's assume that we have already set up a Google Calendar, and we want to share edit access with lots of people.

{{% steps %}}

### Get your Google Calendar ID

After creating the Google Calendar, go to the settings for the calendar and note down the Calendar ID. It will be something like ```lotsofjibberish@group.calendar.google.com```

### Create a Google Form

You're going to share this Google Form link with your trusted people. This Google Form has no questions. It just records the email address that you are currently logged in to. You can enable that in the Google Form settings.

> [!WARNING]
> Check in the Google Form settings that you haven't made the form restrictive to your institution. 
> 
> You want it to be open to anyone with the link, so that people from other institutions can fill it in.

### Write a short Google Apps Script

In the top right of the Google Form, click the "..." button and navigate down to "Script editor". This will open a new tab with the Google Apps Script editor.

We're going to use the following script to add the email address of the person who filled in the form to the Google Calendar as an editor.

```javascript
const CALENDAR_ID = ""; // Get this from the settings in Google Calendar

function grantEditAccess(email_address) {
  return Calendar.Acl.insert(
    {
      'scope': {
        'type': 'user',
        'value': email_address,
      },
      'role': 'writer'
    },
    CALENDAR_ID
  );
}

function onFormSubmit(e) {
  var email_address = e.response.getRespondentEmail();
  grantEditAccess(email_address);
}

function testEditAccess() {
  // You can change this test email address 
  grantEditAccess("some.person@gmail.com");
}
```

You'll need to replace the `CALENDAR_ID` variable with the Calendar ID you noted down earlier.

### Authorise the script

In the left hand side of Google Apps Script, select the "+" next to "Services". Google Calendar v3.

When you run the script for the first time, it will ask you to authorise it. You will need to give it permission to access your Google Calendar and your Google Forms. You only have to do this once. In Google Apps Script, select the `testEditAccess` from the dropdown menu and click the play button (▶️) to run it. This will prompt you to authorise the script.

### Set up the trigger

We want the script to run when someone fills in the form. In Google Apps Script, navigate to the _Triggers_ button on the left (it looks like an alarm clock) and create a trigger. Use the following settings:

- Choose which function to run: `onFormSubmit`
- Select event source: `From form`
- Select event type: `On form submit`


### Share the Google Form

Now you can share the Google Form link with your trusted people. Put it on your Wiki, send it out in your email lists, Slack channels, or whatever. When they fill in the form, it will automatically add them as an editor to the Google Calendar.

{{% /steps %}}

Please add a reaction or comment if you found this useful!