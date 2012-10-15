---
layout: post
title: "Accessing Reminders with EventKit (Part 2)"
date: 2012-10-15 12:05
comments: true
categories: [osx, ios, command-line, Reminders, EventKit]
---

*[Part 1][Part1]*

I want a command-line tool to access Reminders.app data. I was inspired by the command-line password manager, [pass](http://zx2c4.com/projects/password-store). Based on the man page, `pass` uses the following parameters:
{% codeblock lang:bash %}
pass [COMMAND] [OPTIONS]... [ARGS]...
{% endcodeblock %}
This seems like a good model to follow. I'm going to name this command `rem`, short for *reminders*. It also reminds me of my old [BASIC](http://en.wikipedia.org/wiki/BASIC) days, where `REM` started a comment line.

Here's the functionality I want for the first iteration:

  *  `rem [ls] [calendar]` list reminders
  *  `rem rm <calendar> <reminder_id>` remove a reminder
  *  `rem cat <calendar> <reminder_id>` show a reminder details
  *  `rem done <calendar> <reminder_id>` mark a reminder as completed

I'm not including functionality to add or modify a reminder (yet). I just don't need this functionality right now, as I usually add reminders via my iPhone.

Start with the OSX Command Line Tool Application template, with default options of using Foundation and ARC. Add `EventKit.framework` to the `rem` target and away we go.

<!-- more -->

###Parsing the Command-line Arguments###

I need to process the application arguments to know what I want to do. Before doing that, some set up. I `#define` an array of command strings, and an enumerated type that maps to each element of the command string array.

{% codeblock lang:objc %}
#define COMMANDS @[ @"ls", @"rm", @"cat", @"done", @"help", @"version" ]
typedef enum _CommandType {
    CMD_UNKNOWN = -1,
    CMD_LS = 0,
    CMD_RM,
    CMD_CAT,
    CMD_DONE,
    CMD_HELP,
    CMD_VERSION
} CommandType;

static CommandType command;
static NSString *calendar;
static NSString *reminder_id;
{% endcodeblock %}

The three global variables hold the parsed values from the command arguments. Some consider it bad form to use a global like this, but it's just easier to keep it global rather than passing it around.

The function, `parseArguments`, handles the argument parsing and is called from `main`. I won't go over the entire implementation, but want to make note of the of a few things.

{% codeblock lang:objc %}
static void parseArguments()
{
    command = CMD_LS;
    
    NSMutableArray *args = [NSMutableArray arrayWithArray:[[NSProcessInfo processInfo] arguments]];
    [args removeObjectAtIndex:0];    // pop off application argument
    
    // args array is empty, command was excuted without arguments
    if (args.count == 0)
        return;
    
    NSString *cmd = [args objectAtIndex:0];
    command = (CommandType)[COMMANDS indexOfObject:cmd];
    if (command == NSNotFound) {
        printf("rem: Error unknown command %s", [cmd cStringUsingEncoding:NSUTF8StringEncoding]);
        usage();
        exit(-1);
    }
    ...    
}
{% endcodeblock %}

I use [NSProcessInfo](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSProcessInfo_Class/Reference/Reference.html) to get our command-line arguments. The first element will be the application, so I pop that off.

You can see where how the `COMMAND` string array and `CommandType` enumeration have to lineup. I can search the `COMMAND` array for the matching string, and the index with be the CommandType enumeration value. Better than making a giant `if` or `switch` statement. If the command argument isn't found the command string array, then I print an error message and spit out the usage.

The rest of the function reads the reminder list (*calendar*) name and reminder id from the `args` array.

###Loading and Sorting Reminders###

Two functions to handle reminders. `fetchReminders` will load the reminders from the event store. `sortReminders` will organize the fetched reminders into a dictionary keyed on reminder list (calendar) name.

Here's `fetchReminders`.

{% codeblock lang:objc %}
static NSArray* fetchReminders()
{
    __block NSArray *reminders = nil;
    __block BOOL fetching = YES;
    NSPredicate *predicate = [store predicateForRemindersInCalendars:nil];
    [store fetchRemindersMatchingPredicate:predicate completion:^(NSArray *ekReminders) {
        reminders = ekReminders;
        fetching = NO;
    }];

    while (fetching) {
        [[NSRunLoop currentRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:0.01]];
    }
    
    return reminders;
}
{% endcodeblock %}

The `EKEventStore` instance is a global that's instantiated at application start. I use it create a predicate to fetch all the reminders across all calendars. Recall that in [Part 1][Part1], I noted that the event store can be expensive to create and release. For a GUI (Cocoa or iOS), I'd create the event store in the application delegate and have it persist for the lifetime of the app.

I have some funky coding going on here. What's up with the `while` loop after the call to `fetchRemindersMatchingPredicate:completion:`? I have a boolean flag, `fetching`, that gets flipped inside the completion handler block. Until it gets flipped, the application's [runloop](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/Reference/Reference.html) will run for 1/100th of a second. During this 1/100th of second, the completion handler block *may* get invoked. If so, the `fetching` flag will be flipped, and the app will leave the `while` loop.

`fetchRemindersMatchingPredicate:completion:` is an *asynchronous* call. It gets sent to a another thread to run in the background. The problem is that the completion handler block gets invoked back on the main thread which can cause problems. Ideally, I would try to use [Grand Central Dispatch](https://developer.apple.com/library/mac/#documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html) like this:

{% codeblock lang:objc %}
dispatch_semaphore_t sema = dispatch_semaphore_create(0);
[store fetchRemindersMatchingPredicate:predicate completion:^(NSArray *ekReminders) {
    reminders = ekReminders;
    dispatch_semaphore_signal(sema);
}];
dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
{% endcodeblock %}

Since the completion handler block gets invoked on the main thread, the semaphone blocks, and the signal is never sent. So looping over the run loop seems like the best bet.

**Note:** *This is also the reason I'm loading all the reminders, then sorting them. If I had access to a synchronous reminder fetch, my first instinct would be to loop over each reminder list. However, loading all at once, then sorting is probably faster.*

Now that I have an array of reminders, I sort them into a dictionary, keyed on calendar names. I ignore the completed reminders.

{% codeblock lang:objc %}
static NSDictionary* sortReminders(NSArray *reminders)
{
    NSMutableDictionary *results = nil;
    if (reminders != nil && reminders.count > 0) {
        results = [NSMutableDictionary dictionary];
        for (EKReminder *r in reminders) {
            if (r.completed)
                continue;
            
            EKCalendar *calendar = [r calendar];
            if ([results objectForKey:calendar.title] == nil) {
                [results setObject:[NSMutableArray array] forKey:calendar.title];
            }
            NSMutableArray *calendarReminders = [results objectForKey:calendar.title];
            [calendarReminders addObject:r];
        }
    }
    return results;
}
{% endcodeblock %}

###Handling the Command###

I've got my command-line arguments, and I've got my (sorted) reminders. Now we need to do something. `handleCommand` is a function around a switch statement for command-specific functions. We'll just stub out those functions right now.

{% codeblock lang:objc %}
static void listReminders() 
{
    NSLog(@"List Reminders"); 
}

static void removeReminder()
{
    NSLog(@"Remove Reminders %@/%@", calendar, reminder_id); 
}

static void showReminder()
{
    NSLog(@"Show Reminders %@/%@", calendar, reminder_id); 
}

static void completeReminder()
{
    NSLog(@"Complete Reminders %@/%@", calendar, reminder_id); 
}

static void handleCommand()
{
    switch (command) {
        case CMD_LS:
            listReminders();
            break;
        case CMD_RM:
            removeReminder();
            break;
        case CMD_CAT:
            showReminder();
            break;
        case CMD_DONE:
            completeReminder();
            break;
        case CMD_HELP:
        case CMD_VERSION:
            break;
    }

}
{% endcodeblock %}

At this point, I can build `rem` and run it. Each command I defined goes to the function that should handle it. But we're not printing anything out yet. I'll get to that in Part 3.

[Part1]: /blog/2012/10/09/accessing-reminders-with-eventkit-part-1/
[Repo]: https://github.com/kykim/rem