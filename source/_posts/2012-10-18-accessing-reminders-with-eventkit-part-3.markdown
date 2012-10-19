---
layout: post
title: "Accessing Reminders with EventKit (Part 3)"
date: 2012-10-18 11:37
comments: true
categories: [osx, ios, command-line, Reminders, EventKit]
---

*[Part 1][Part1], [Part 2][Part2]*

I left off with the `rem` command-line app parsing the command-line arguments and stubbed out the functions to handle each command. I'm going to finish up the app by fleshing out each command function.

Before working on the handler functions, I want to do one more thing. I want to check that my command-line arguments actually specify an actual reminder list (calendar) and reminder id (index).

<!-- more -->

The `EKCalendarItem` has two identifier properties: `calendarItemIdentifier` and `calendarItemExternalIdentifier`. At first thought, it would be a good idea to use one of those properties as the reminder id. However, both identifier properties return a [GUID](http://en.wikipedia.org/wiki/Globally_unique_identifier), which is great for programming purposes, but not so ideal for a command-line app. So, reminder id will be a simple integer to represent the reminder position in the reminder list.

Basically, we want to check the reminder list  name specified and compare it to the known names in the `calendars` dictionary. If the name is valid, then we check to see if the specified reminder id is within the index range of the reminder array for a given reminder list.

{% codeblock lang:objc %}
static void validateArguments()
{
    if (command == CMD_LS && calendar == nil)
        return;
    
    NSUInteger calendar_id = [[calendars allKeys] indexOfObject:calendar];
    if (calendar_id == NSNotFound) {
        _print(stderr, @"rem: Error - Unknown Reminder List: \"%@\"\n", calendar);
        exit(-1);
    }
    
    if (command == CMD_LS && reminder_id == nil)
        return;
    
    NSInteger r_id = [reminder_id integerValue] - 1;
    NSArray *reminders = [calendars objectForKey:calendar];
    if (r_id < 0 || r_id > reminders.count-1) {
        _print(stderr, @"rem: Error - ID Out of Range for Reminder List: %@\n", calendar);
        exit(-1);
    }
    reminder = [reminders objectAtIndex:r_id];
}
{% endcodeblock %}

The `ls` command allows the `calendar` and `reminder_id` variables to be `nil`. Other than that, we use the arguments to assign values to the variables `calendar` and `reminder`.

###Displaying Reminders###

Before implementing each function, I need some special characters for displaying the reminders. Let's take another look at the [pass][Pass] output:

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

It turns out that `pass` uses the file system to store everything. To get the nicer output, it uses a program called [tree][Tree], which prints out file-system hierarchies. We could hack something that writes out to the file-system, then use `tree` to display it, but that's overkill. Our reminders aren't stored in a tree; they're only two levels deep: the reminders and their list.

Looking at the `pass` output, there are three "special" strings: "│   ", "├──", "└──". Looking at the `tree` source, I found the unicode encodings for them. But, I'm cheating and just cutting and pasting into the code.

{% codeblock lang:objc %}
#define TACKER @"├──"
#define CORNER @"└──"
#define PIPER  @"│  "
#define SPACER @"   "
{% endcodeblock %}

Now, there are two things to consider:
  1.  The last calender and reminder need to use the corner character.
  2.  The reminders of the last calendar don't have the vertical bar.
For expediency's sake, I'm using two functions for output. One for calendars and one for reminders.

{% codeblock lang:objc %}
static void _printCalendarLine(NSString *line, BOOL last)
{
    NSString *prefix = (last) ? CORNER : TACKER;
    _print(stdout, @"%@ %@\n", prefix, line);
}

static void _printReminderLine(NSString *line, BOOL last, BOOL lastCalendar)
{
    NSString *indent = (lastCalendar) ? SPACER : PIPER;
    NSString *prefix = (last) ? CORNER : TACKER;
    _print(stdout, @"%@%@ %@\n", indent, prefix, line);
}
{% endcodeblock %}

###Listing Reminders###

Now, the `listReminders` function is suppose to handle two cases: reminders in a specific reminder list or all reminders. The actually calls to `printCalendarLine` and `printReminderLine` will be handled by `listCalendar`.

{% codeblock lang:objc %}
static void listCalendar(NSString *cal, BOOL last)
{
    printCalendarLine(cal, last);
    NSArray *reminders = [calendars valueForKey:cal];
    for (EKReminder *r in reminders) {
        printReminderLine(r.title, (r == [reminders lastObject]), last);
    }
}

static void listReminders()
{
    printf("Reminders\n");
    if (calendar) {
        listCalendar(calendar, YES);
    }
    else {
        for (NSString *cal in calendars) {
            listCalendar(cal, (cal == [[calendars allKeys] lastObject]));
        }
    }
}
{% endcodeblock %}

Trying it out, it gives us this output.

{% codeblock lang:bash %}
kykim$ rem
Reminders
├── Work
│   ├── Blog about EventKit
│   └── Finish More iOS Programming Book
└── Home
    ├── Pay Electric
    └── Feed Cat

kykim$ ./rem ls Home
Reminders
└── Home
    ├── Pay Electric
    └── Feed Cat
{% endcodeblock %}

Looks good, but one more tweak. All the other commands use a reminder_id to identify which reminder to use. Recall, that I'm just printing a simple integer index as the reminder_id. `printReminderLine` and `listCalendar` change to this.

{% codeblock lang:objc %}
static void _printReminderLine(NSUInteger id, NSString *line, BOOL last, BOOL lastCalendar)
{
    NSString *indent = (lastCalendar) ? SPACER : PIPER;
    NSString *prefix = (last) ? CORNER : TACKER;
    _print(stdout, @"%@%@ %ld. %@\n", indent, prefix, id, line);
}

static void _listCalendar(NSString *cal, BOOL last)
{
    _printCalendarLine(cal, last);
    NSArray *reminders = [calendars valueForKey:cal];
    for (NSUInteger i = 0; i < reminders.count; i++) {
        EKReminder *r = [reminders objectAtIndex:i];
        _printReminderLine(i+1, r.title, (r == [reminders lastObject]), last);
    }
}
{% endcodeblock %}

###Removing a Reminder###

Removing a reminder is a simple call to `removeReminder:commit:error:` to the event store.

{% codeblock lang:objc %}
static void removeReminder()
{
    NSError *error;
    BOOL success = [store removeReminder:reminder commit:YES error:&error];
    if (!success) {
        _print(stderr, @"rem: Error removing Reminder (%@) from list %@\n\t%@", reminder_id, calendar, [error localizedDescription]);
    }
}
{% endcodeblock %}

###Show a Reminder Details###

Showing a reminder is a little verbose, but pretty straight-forward. The only catch is using an `NSDateFormatter` instance to display the date properties of a reminder. The reminder properties I chose to display are: title, calendar (reminder list) name, creation date, last modification date (if different than creation date), start date (if set), due date (if set), and notes (if set).

{% codeblock lang:objc %}
static void showReminder()
{
    _print(stdout, @"Reminder: %@\n", reminder.title);
    _print(stdout, @"\tList: %@\n", calendar);
    
    _print(stdout, @"\tCreated On: %@\n", [dateFormatter stringFromDate:reminder.creationDate]);
        
    if (reminder.lastModifiedDate != reminder.creationDate) {
        _print(stdout, @"\tLast Modified On: %@\n", [dateFormatter stringFromDate:reminder.lastModifiedDate]);
    }
        
    NSDate *startDate = [reminder.startDateComponents date];
    if (startDate) {
        _print(stdout, @"\tStarted On: %@\n", [dateFormatter stringFromDate:startDate]);
    }
        
    NSDate *dueDate = [reminder.dueDateComponents date];
    if (dueDate) {
        _print(stdout, @"\tDue On: %@\n", [dateFormatter stringFromDate:dueDate]);
    }
    
    if (reminder.hasNotes) {
        _print(stdout, @"\tNotes: %@\n", reminder.notes);
    }
}
{% endcodeblock %}

###Completing a Reminder###

Completing reminder is a simple matter of setting the completed property to YES and saving it.

{% codeblock lang:objc %}
static void completeReminder()
{
    reminder.completed = YES;
    NSError *error;
    BOOL success = [store saveReminder:reminder commit:YES error:&error];
    if (!success) {
        _print(stderr, @"rem: Error marking Reminder (%@) from list %@\n\t%@", reminder_id, calendar, [error localizedDescription]);
    }
}
{% endcodeblock %}

That's it. Now I have a simple command-line utility to let me list, view, remove and complete reminders. The project is up on [Github][Repo]. Feel free to use and give me your feedback. I have plans to extend `rem`, check the [issues](https://github.com/kykim/rem/issues) for what's in the queue.

[Part1]: /blog/2012/10/09/accessing-reminders-with-eventkit-part-1/
[Part2]: /blog/2012/10/15/accessing-reminders-with-eventkit-part-2/
[Repo]: https://github.com/kykim/rem
[Pass]: http://zx2c4.com/projects/password-store
[Tree]: http://mama.indstate.edu/users/ice/tree/
