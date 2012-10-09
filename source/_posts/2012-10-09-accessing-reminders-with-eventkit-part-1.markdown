---
layout: post
title: "Accessing Reminders with EventKit (part 1)"
date: 2012-10-09 15:56
comments: true
categories: [osx, ios, command-line, Reminders, EventKit]
---

I'm old school when it comes to certain things. When the [Titanium Powerbook G4](http://en.wikipedia.org/wiki/PowerBook_G4#Titanium_PowerBook_G4) came out, I was excited because I could have six Terminal windows open at once. I still use [Emacs](http://www.gnu.org/software/emacs/). My biggest complaint about Xcode 4 is they got rid of the Emacs client integration.

On the other hand, sometimes newer things work for me as well. When Reminders came out in iOS 5, I started using it, but wasn't diligent about it. Once it got [iCloud and Mountain Lion integration](http://www.apple.com/osx/whats-new/features.html#reminders), I started using it a lot more. I even use [Siri](http://www.apple.com/ios/whats-new/#siri) to make a reminder. I make a reminder on my iPhone, and I have it on all my devices, so I actually keep track of things.

Recently, I started using the command line password manager, [pass](http://zx2c4.com/projects/password-store), to manage my passwords. I tried other password managers, and this is a case where the simpler command-line interface works for me. You type `pass` and get a list of all your accounts and passwords, which you catagorize. There are simple commands to add, extract and delete passwords. It's all encrypted using [GPG](http://www.gnupg.org), so I don't need to worry if I lose my laptop. All the data is stored in my home directory, so it's part of my [Time Machine](http://www.apple.com/osx/whats-new/features.html#timemachine) backups.

{% codeblock pass lang:bash %}
zx2c4@laptop ~ $ pass
Password Store
├── Business
│   ├── some-silly-business-site.com
│   └── another-business-site.net
├── Email
│   ├── donenfeld.com
│   └── zx2c4.com
└── France
    ├── bank
    ├── freebox
    └── Mobilephone
{% endcodeblock %}

I wanted some similar for to access reminders. Now Apple's been pretty good about making command-line equivalents for a lot of UI tools (think DiskUtility.app and diskutil), but this is one where they don't have anything. A quick Google didn't really find anything for me either.

So I have roll up my sleeves and write something myself.

<!-- more -->

Turns out that along with releasing Reminders in Mountain Lion, Apple has given us an SDK to access the data store behind Calendar and Reminders: **EventKit**. [EventKit][EK] was first introduced in the OSX and iOS SDKs to give access to Calendar. Now with OSX Mountain Lion and iOS 6, Apple extended EventKit to grant access to Reminders. This extension to EventKit was possible because underneath it all, items in Reminders (“*reminders*”) are an extension of events in Calendar (“*events*”). In fact, both sets of data are stored in a single database, which Apple calls the Calendar database.

Why not use something like [SQLite](http://sqlite.org) and let us access the calendar database directly? Odds are that your local calendar(s) are synchronized with calendars on server (i.e. iCloud, gCal, etc), typically using [CalDAV](http://caldav.calconnect.org). In order to maintain that synchronization, it’s easier to mediate all access to the local Calendar database to know when changes need to propagated to the server and known when changes have arrived from the server.

**Note:** *I’m building an application to run in OSX. Most of what I cover is applicable for iOS, but there are some differences. For example, instantiating the event store is different between OSX and iOS. Read the [Calendar and Reminders Programming Guide][EK]. Also, I’m building an application to access reminder, I won’t be covering event access.*

Let's review the EventKit framework with respect to reminders.

###Event Store###
At the root of EventKit is the class `EKEventStore`. This represents the Calendar database. `EKEventStore` is a fairly heavy-weight object, taking a (relatively) long time to instantiate and release. As a result, your application should only instantiate a single `EKEventStore`.

To instantiate your event store, you invoke the initializer, `initWithAccessToEntityType:`, and `EKEntityMask` for reminders.

{% codeblock lang:objc %}
EKEventStore *store = [[EKEventStore alloc] initWithAccessToEntityType:EKEntityTypeReminder];
{% endcodeblock %}

With iOS, you instantiate your event store with a simple call to `init`. Then, you request access to reminders using `requestAccessToEntityType:completion:`. This invoking this method will cause iOS to ask the user if your application is allowed to access your Calendar database. You need to handle both cases in the completion block. This code is not in the OSX EventKit, since access to the Calendar database is granted automatically.

{% codeblock lang:objc %}
EKEventStore *store = [EKEventStore alloc] init];

[store requestAccessToEntityType:EKEntityTypeReminder completion:^(BOOL granted, NSError *error) {
    // access code here
}];
{% endcodeblock %}

###Reminder Lists###

EventKit defines the class `EKCalendar` to represent a calendar. Events are represented with the class `EKEvent`. Reminders are modeled with the `EKReminder` class. `EKEvent` and `EKReminder` are extensions of the abstract class `EKCalendarItem`. Rather than defining a new class to contain a list of reminders, Apple chose to leverage the `EKCalendar` class as the container of reminders. Conceptually, Reminders calls these Reminder Lists, but internal to the event store, they are just instances of `EKCalendar`.

`EKEventStore` defines two properties to access default the calendar and reminder list.

{% codeblock lang:objc %}
EKCalendar *defaultCalendar = [store defaultCalendarForNewEvents];
EKCalendar *defaultReminderList = [store defaultCalendarForNewReminders];
{% endcodeblock %}

If you want to retrieve all calendars or reminder lists in the event store, you use the `calendarsForEntityType:` method, specifying the `EKEntityType` you want.

{% codeblock lang:objc %}
NSArray *calendars = [store calendarsForEntityType:EKEntityTypeEvent];
NSArray *reminderLists = [store calendarsForEntityType:EKEntityTypeReminder];
{% endcodeblock %}

There’s another method, `calendarWithIdentifier:`, that fetches a specific calendar, assuming you know the calendar’s unique identifier.

###Reminders###

To create a new reminder, use the `EKReminder` class method `reminderWithEventStore:`. In order for the reminder to be valid, you need to provide values for the `title` and `calendar` properties.

{% codeblock lang:objc %}
EKReminder *new_reminder = [EKReminder reminderWithEventStore:store];
new_reminder.title = @“New Reminder”;
new_reminder.calendar = [store defaultCalendarForNewReminders];
{% endcodeblock %}

To specify a start date, you use the `startDateComponents` property. To specify a due date, use the `dueDateComponents` property. These two properties are of the `NSDateComponents`([doc](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSDateComponents_Class/Reference/Reference.html)) class, **not** the `NSDate` class.

There are two EKReminder properties associated with reminder completion:

  *  `completed` a `BOOL` flag
  *  `completionDate` an `NSDate` instance marking when the reminder was completed.

Setting on of these properties will adjust the other. For example, if you set `completed` to `YES`, then `completionDate` will be set to the current date. Setting `completed` to `NO`, sets `completionDate` to `nil`. Conversely, setting `completionDate` to a date will set `completed` to `YES`; and setting `completionDate` to `nil` will set `completed` to `NO`.

To save a reminder, call the `saveReminder:commit:error:` method. To delete a reminder, use the `removeReminder:commit:error:` method.

{% codeblock lang:objc %}
NSError *error;
BOOL success = [store saveReminder:reminder commit:YES error:&error];
if (!success) {
    // Handle failure here, look at error instance
}
{% endcodeblock %}

{% codeblock lang:objc %}
NSError *error;
BOOL success = [store removeReminder:reminder commit:YES error:&error];
if (!success) {
    // Handle failure here, look at error instance
}
{% endcodeblock %}

In both cases, the `commit` flag is used to tell the event store whether to apply the save/delete immediately, or queue the action as part of a batch. If you pass `commit:NO`, then the changes will not happen until you invoke the EKEventStore method, `commit:`.

{% codeblock lang:objc %}
NSError *error;
BOOL success = [store commit:&error];
if (!success) {
    // Handle failure here, look at error instance
}
{% endcodeblock %}

###Retrieving Reminders###

To retrieve your reminders from the event store, you use the `fetchRemindersMatchingPredicate:completion:` method. Predicates are objects that encapsulate the conditions to perform a search or query. Even though we use an `NSPredicate`, we can’t construct our EventKit predicate by hand. We have to use one of the three predicate construction methods on `EKEventStore`.

  *  `predicateForIncompleteRemindersWithDueDateStarting:ending:calendars:` find all incomplete Reminder items within a date range, for a given array of Reminder Lists.
  *  `predicateForCompletedRemindersWithCompletionDateStarting:ending:calendars:` find all completed Reminder items within a date rage for a given array of Reminder Lists.
  *  `predicateForRemindersInCalendars:` find all Reminder items for a given array of Reminder Lists.

To fetch across all reminder lists (calendars), you can simply specify `nil` for the calendar array.

{% codeblock lang:objc %}
NSPredicate *predicate = [store predicateForRemindersInCalendars:nil];
{% endcodeblock %}

For the predicate generator methods that take date ranges, you can use the `NSDate` class methods `distantPast` and `distantFuture` to fetch all reminder items.

{% codeblock lang:objc %}
NSPredicate *predicate = [store predicateForIncompleteRemindersWithDueDateStarting:[NSDate distantPast] ending:[NSDate distantFuture] calendars:nil];
{% endcodeblock %}

Once you have your predicate, you can perform your fetch, with a completion handler block.

{% codeblock lang:objc %}
[store fetchRemindersMatchingPredicate:predicate completion:^(NSArray *reminders) {
    for (EKReminder *reminder in reminders) {
        // Process each reminder here
    }
}];
{% endcodeblock %}

The completion handler block is send an array of reminder items that match the predicate. This fetch is performed asynchronously and doesn’t need to be dispatched to another thread. However, the completion handler block is invoked on the main thread, which caused problems for me (I’ll get to that in a little bit).
As with calendars, there is a method to retrieve a specific reminder if you know its unique identifier, `calendarItemWithIdentifier:`. You probably won’t use this method as it’s unlikely that you’ll have an item’s unique identifier handy.

###Alarms, Recurrence, and External Changes###

I’m not going to cover these features in any detail, since I don’t need them for my command-line application. I’ve provided some links to the Apple documentation.

  *  [Alarms](https://developer.apple.com/library/mac/#documentation/DataManagement/Conceptual/EventKitProgGuide/ConfiguringAlarms/ConfiguringAlarms.html): You can set an alarm for a given reminder, which can be time or location based.
  *  [Recurrence](https://developer.apple.com/library/mac/#documentation/DataManagement/Conceptual/EventKitProgGuide/CreatingRecurringEvents/CreatingRecurringEvents.html): You can set a reminder to repeat.
  *  [External Changes](https://developer.apple.com/library/mac/#documentation/DataManagement/Conceptual/EventKitProgGuide/ObservingChanges/ObservingChanges.html): As I mentioned earlier, odds are your calendars are tied to a server somewhere. As a result changes from the outside need to be reflected in you EventKit application.

###Coming Up: Building Our Command-line Reminders Tool###

Ok, that's a pretty good review of the EventKit framework. My next post will use this information to build the command-line tool.

[EK]: https://developer.apple.com/library/mac/#documentation/DataManagement/Conceptual/EventKitProgGuide “Calendar and Reminders Programming Guide”

