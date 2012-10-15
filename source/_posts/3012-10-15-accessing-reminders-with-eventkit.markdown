---
layout: post
title: "Accessing Reminders with EventKit"
date: 3012-10-15 00:00
comments: true
categories: [osx, ios, command-line, Reminders, EventKit]
---

Our `main` function is very simple: parse the arguments; load and sort the reminders; then do something. We're almost at a good stopping point, so let's get something out of this.

{% codeblock lang:objc %}
static void listReminders()
{
    printf("Reminders\n");
    for (NSString *cal in calendars) {
        printf("%s\n", [cal cStringUsingEncoding:NSUTF8StringEncoding]);
        for (EKReminder *r in [calendars valueForKey:cal]) {
            printf("\t%s\n", [r.title cStringUsingEncoding:NSUTF8StringEncoding]);
        }
    }
}
{% endcodeblock %}

When I build and run the app without any arguments, I get my incomplete reminders listed.

{% codeblock lang:bash %}
Reminders
Work
	Blog about EventKit
	Finish More iOS Programming Book
Home
	Pay Electric
	Feed Cat
{% endcodeblock %}

As a matter of fact, I get the same list even if I type `rem ls Home`. All I'm doing is list all the reminders.

When we left off, we had gotten our command-line app, `rem`, to list out all our reminders organized by reminder list (calendar). Let's clean up the code so the output is a little nicer. Our inspiration for this was the command-line password manager, [pass](http://zx2c4.com/projects/password-store). The output from `pass` looks like this.

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

Right now, the output from `rem` looks like this.

{% codeblock lang:bash %}
Reminders
Work
        Blog about EventKit
        Finish More iOS Programming Book
Home
        Pay Electric
        Feed Cat
{% endcodeblock %}

It turns out that `pass` uses the file system to store everything. To get the nicer output, it uses a program called (tree)[http://mama.indstate.edu/users/ice/tree/], which prints out file-system hierarchies. We could hack something that writes out to the file-system, then use `tree` to display it, but that's overkill. Our reminders aren't stored in a tree; they're only two levels deep: the reminders and their list.

Looking at the `pass` output, there are three "special" characters we need, the vertical line, the spacer (like a sideways 'T'), and the corner (like a backwards, sideways 'L'). The vertical line is on our keyboard. The other two are a little more difficult. But, I cheated and found the UTF-8 encoding in the `tree` source. I added them to `rem` as `#defines`.

{% codeblock lang:objc %}
#define SPACER "\342\224\234\342\224\200\342\224\200"
#define CORNER "\342\224\224\342\224\200\342\224\200"
{% endcodeblock %}

Now, there are two things we have to consider:
  1.  The last calender and reminder need to use the corner character.
  2.  The reminders of the last calendar don't have the vertical bar.
For expediency's sake, there are two functions for output. One for calendars and one for reminders.

{% codeblock lang:objc %}
static void printCalendarLine(NSString *line, BOOL last)
{
    const char *prefix = (last) ? CORNER : SPACER;
    printf("%s %s\n", prefix, [line cStringUsingEncoding:NSUTF8StringEncoding]);
}

static void printReminderLine(NSString *line, BOOL last, BOOL lastCalendar)
{
    const char *indent = (lastCalendar) ? "    " : "|   ";
    const char *prefix = (last) ? CORNER : SPACER;
    printf("%s%s %s\n", indent, prefix, [line cStringUsingEncoding:NSUTF8StringEncoding]);
}
{% endcodeblock %}

Now, the `listReminders` function is suppose to handle two cases: specific reminder list or all reminders. The actually calls to `printCalendarLine` and `printReminderLine` will be handled by `listCalendar`.

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

Now, the output looks like this.

{% codeblock lang:objc %}
kykim$ ./rem
Reminders
├── Work
|   ├── Blog about EventKit
|   └── Finish More iOS Programming Book
└── Home
    ├── Pay Electric
    └── Feed Cat
kykim$
kykim$ ./rem ls Home
Reminders
└── Home
    ├── Pay Electric
    └── Feed Cat
{% endcodeblock %}

Looking good, but one more tweak. All the other commands use a reminder_id to identify which reminder to use. The identifier properties return a (GUID)[http://en.wikipedia.org/wiki/Globally_unique_identifier], which is great for programming purposes, but not so ideal for a command-line app. I just a simple integer index as the reminder_id. `printReminderLine` and `listCalendar` change to this.

{% codeblock lang:objc %}
static void printReminderLine(NSUInteger id, NSString *line, BOOL last, BOOL lastCalendar)
{
    const char *indent = (lastCalendar) ? "    " : "|   ";
    const char *prefix = (last) ? CORNER : SPACER;
    printf("%s%s %ld. %s\n", indent, prefix, id, [line cStringUsingEncoding:NSUTF8StringEncoding]);
}

static void listCalendar(NSString *cal, BOOL last)
{
    printCalendarLine(cal, last);
    NSArray *reminders = [calendars valueForKey:cal];
    for (NSUInteger i = 0; i < reminders.count; i++) {
        EKReminder *r = [reminders objectAtIndex:i];
        printReminderLine(i+1, r.title, (r == [reminders lastObject]), last);
    }
}
{% endcodeblock %}

###Removing a Reminder###

{% codeblock lang:objc %}
static void removeReminder()
{
    NSArray *reminders = [calendars objectForKey:calendar];
    NSInteger r_id = [reminder_id integerValue]-1;
    if (r_id < 0 || r_id > reminders.count-1) {
        printf("rem: Error - ID Out of Range for Reminder List %s", [calendar cStringUsingEncoding:NSUTF8StringEncoding]);
    }
    EKReminder *reminder = [reminders objectAtIndex:r_id];
    NSError *error;
    BOOL success = [store removeReminder:reminder commit:YES error:&error];
    if (!success) {
        NSString *errorMessage = [NSString stringWithFormat:@"rem: Error removing Reminder (%@) from list %@\n\t%@", reminder_id, calendar, [error localizedDescription]];
        printf("%s", [errorMessage cStringUsingEncoding:NSUTF8StringEncoding]);
    }
}
{% endcodeblock %}

###Show a Reminder Details###

{% codeblock lang:objc %}
static void showReminder()
{
    NSArray *reminders = [calendars objectForKey:calendar];
    NSInteger r_id = [reminder_id integerValue]-1;
    if (r_id < 0 || r_id > reminders.count-1) {
        printf("rem: Error - ID Out of Range for Reminder List %s\n", [calendar cStringUsingEncoding:NSUTF8StringEncoding]);
    }
    EKReminder *reminder = [reminders objectAtIndex:r_id];

    printf("Reminder: %s\n", [reminder.title cStringUsingEncoding:NSUTF8StringEncoding]);
    printf("\tList: %s\n", [calendar cStringUsingEncoding:NSUTF8StringEncoding]);
    
    if (!reminder.completed) {
        printf("\tCreated On: %s\n", [[dateFormatter stringFromDate:reminder.creationDate] cStringUsingEncoding:NSUTF8StringEncoding]);
        
        if (reminder.lastModifiedDate != reminder.creationDate) {
            printf("\tLast Modified On: %s\n", [[dateFormatter stringFromDate:reminder.lastModifiedDate] cStringUsingEncoding:NSUTF8StringEncoding]);
        }
        
        NSDate *startDate = [reminder.startDateComponents date];
        if (startDate) {
            printf("\tStarted On: %s\n", [[dateFormatter stringFromDate:startDate] cStringUsingEncoding:NSUTF8StringEncoding]);
        }
        
        NSDate *dueDate = [reminder.dueDateComponents date];
        if (dueDate) {
            printf("\tDue On: %s\n", [[dateFormatter stringFromDate:dueDate] cStringUsingEncoding:NSUTF8StringEncoding]);
        }
    }
    else {
        printf("\tCompleted On: %s\n", [[dateFormatter stringFromDate:reminder.completionDate] cStringUsingEncoding:NSUTF8StringEncoding]);
    }
    
    if (reminder.hasNotes) {
        printf("\tNotes: %s\n", [reminder.notes cStringUsingEncoding:NSUTF8StringEncoding]);
    }
}
{% endcodeblock %}

###Completing a Reminder###

{% codeblock lang:objc %}
static void completeReminder()
{
    NSArray *reminders = [calendars objectForKey:calendar];
    NSInteger r_id = [reminder_id integerValue]-1;
    if (r_id < 0 || r_id > reminders.count-1) {
        printf("rem: Error - ID Out of Range for Reminder List %s\n", [calendar cStringUsingEncoding:NSUTF8StringEncoding]);
    }
    EKReminder *reminder = [reminders objectAtIndex:r_id];

    reminder.completed = YES;
    NSError *error;
    BOOL success = [store saveReminder:reminder commit:YES error:&error];
    if (!success) {
        NSString *errorMessage = [NSString stringWithFormat:@"rem: Error marking Reminder (%@) from list %@\n\t%@", reminder_id, calendar, [error localizedDescription]];
        printf("%s", [errorMessage cStringUsingEncoding:NSUTF8StringEncoding]);
    }
}
{% endcodeblock %}
