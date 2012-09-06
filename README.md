# iOS Integration Guide for iPhone and iPad
1. **[Upgrading](#upgrading)**
2. **[Minimum Library Version](#all-apps-must-run-version-211-or-above)**
3. **[UDID removed for iOS 6](#udid-removed-for-ios-6)**
4. **[iOS ARC Support](#ios-arc-support)**
5. **[Instructions](#ten-minute-instrumentation-instructions)**
6. **[Screen Lock](#screen-lock-optional)**
7. **[Events (Optional)](#events-optional)**
8. **[Screen Flows (Premium Feature)](#screen-flows-premium-feature)**
9. **[Sample Application Skeleton](#sample-application-skeleton)**
10. **[Debugging No Data](#debugging-no-datatroubleshooting)**
11. **[Deprecated Libraries](#deprecated-libraries)**

## Upgrading
If you are upgrading from an older version of the SDK, simply replace the Localytics source files in your project with the ones you downloaded. If the version you are upgrading from did not use sqlite and libz, skip to step 11. If you get an error about an unrecognized selector it means you need to clean your project and rebuild.

## All Apps Must Run Version 2.11 or Above
All apps submitted to the App Store must be running the version 2.11 or greater of the Localytics iOS SDK to ensure they are approved by Apple. This library is fully compatible with iOS 6 and previous iOS versions. The target platform has to be at least iOS 4.

## UDID Removed for iOS 6
Localytics now uses the new identifierForAdvertising provided by Apple in iOS 6 to ensure that anonymous unique users, retention reports and other analysis are accurate and consistent for all iPhone and iPad apps. The Unique Device Identifier (UDID), which was deprecated but still available in iOS 5, is no longer used in iOS 6. This transition will be managed by Localytics to avoid the appearance of “new” users resulting from upgrades to iOS 6.

## iOS ARC Support
The Localytics library manages its own memory manually and is not configured for the new Automatic Reference Counting (ARC) capabilities of iOS 4 and later. It may be necessary to exclude the Localytics source from ARC by setting the -fno-objc-arc compiler flag in the Compile Sources for LocalyticsDatabase.m, LocalyticsSession.m and UploaderThread.m.

See this screenshot for an example.

## Ten Minute Instrumentation Instructions
1. Log into your Localytics account or create an account if you do not already have one.
2. In your account, create a new application and copy the application key.
3. Download the iOS Library as Source.
4. Copy the entire ‘src’ folder from the source package, which you downloaded in step 3 to your application’s project directory.
5. Open your project in Xcode and right click on the project folder in the project navigator and select ‘Add files to <project name>’. Select all the files from the src folder (Step 4) and click Ok.
6. In your application’s delegate file, the file whose default name is <YourApplication>AppDelegate.m, add the following line under any existing imports:

```objective-c
#import "LocalyticsSession.h"
```

7. In the same file, add the following line to the start of applicationDidFinishLaunching. This opens the session and causes an upload on app start. If you intend to track screen flow this has to happen before thew rootViewController is changed. In newer projects the function you are adding to may be called didFinishLaunchingWithOptions. If you do not have this, or any other application functions defined, simply create them from the definitions below:

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
	// Override point for customization after application launch.
	[[LocalyticsSession sharedLocalyticsSession] startSession:@"APP KEY FROM STEP 2"];
 
	self.window.rootViewController = self.viewController;
	[self.window makeKeyAndVisible];

	return YES;
}
```

TIP: Once data is uploaded it cannot be deleted. Therefore it is recommended to create a test app in Localytics and use that while you perfect your instrumentation and then switch the app key for the production app.
8. In the same file, add the following lines to the end of applicationDidEnterBackground. This closes the session when the app goes into the background and attempts an upload.

```objective-c
- (void)applicationDidEnterBackground:(UIApplication *)application {
	[[LocalyticsSession sharedLocalyticsSession] close];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
```

9. In the same file, add the following lines to the end of applicationWillEnterForeground. This attempts to resume the previous session or create a new one if more than 15 seconds have passed and upload any data (nothing happens if data was successfully uploaded by the applicationDidEnterBackground call).

```objective-c
- (void)applicationWillEnterForeground:(UIApplication *)application {
	[[LocalyticsSession sharedLocalyticsSession] resume];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
```

10. In the same file, add the following lines to ApplicationWillTerminate. Under normal circumstances this function will not be called but there are some cases where the OS will terminate the app.

```objective-c
- (void)applicationWillTerminate:(UIApplication *)application {
	// Close Localytics Session
	[[LocalyticsSession sharedLocalyticsSession] close];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
```

11. Add sqlite and libz if they are not already part of your project:
Click on the project in the project navigator. This will bring up the project view.
From here, select your target and then select the ‘Build Phases’ tab.
Open the ‘Link Binaries with Libraries’ expander and click the’+’ button. Search for libz and select ‘libz.dylib’ (it may be in a folder) and click ‘add’.
Repeat the above to search for libsqlite3 to add ‘libsqlite3.dylib’.

12. Test your app. Launch the simulator (or even better, a real device), let the data upload, and view it on the webservice to make sure it is there. Remember that the upload is only guaranteed when the session is started so any events you tag may not appear until your second session.

## Screen Lock (Optional)
When the screen locks due to idle usage the app is not stopped. As a result, with the above integration time spent on a locked screen contributes to session length. When the screen locks applicationWillResignActive is called by the app delegate, and when the user returns applicationDidBecomeActive is called. In this calls the session can be closed and resumed. This way, if the screen is locked for more than 15 seconds the session is ended and a new session is created when the user comes back:

```objective-c
- (void)applicationWillResignActive:(UIApplication *)application
{
	[[LocalyticsSession sharedLocalyticsSession] close];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
 
- (void)applicationDidBecomeActive:(UIApplication *)application
{
	[[LocalyticsSession sharedLocalyticsSession] resume];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
```

## Events (Optional)
Anywhere in your application where an interesting event occurs you may tag it by adding the following line of code:

```objective-c
[[LocalyticsSession sharedLocalyticsSession] tagEvent:@"Interesting Event"];
```

where “Interesting Event” is a string describing the event. It is recommended that you read the Tagging section in the Developer’s Integration Guide in order to get the most value out of your tags. For some events, it may be interesting to collect additional data about the event. Such as how many lives the player has, or what the last action the user took was before clicking on an advertisement. This is accomplished with the second form of tagEvent, which takes a dictionary of key/value pairs along with the event name:

```objective-c
NSDictionary *dictionary =
[NSDictionary dictionaryWithObjectsAndKeys:@"miles per hour", @"display units", @"yes", @"blank screen", nil];
[[LocalyticsSession sharedLocalyticsSession] tagEvent:@"Options saved" attributes:dictionary];
```

See our introductory blog post for more information. It is recommended that you read the Event Attributes section in the Developer’s Integration Guide in order to get the most value out of your attributes.

Important Note:
Make sure never to upload data from a continuous set or from user-generated values. Consider tagging a file transfer event with the file size and name. If the file can be any size this will cause many buckets to be created on our server. When you look at the data in the charts, there will be many charts each with only a few results and the data will be in-actionable. Instead, it is necessary to bucket this data into groups such as “0 to 1K”, “1K to 50K”, etc. Similarly, if you allow users to create their own name for the file, you will have a list of thousands of unique file names that are also not useful to view in the dashboard. See our blogpost for an example.

## Screen Flows (Premium Feature)
When a view is shown, it can be tagged so that all user flows through the application can be tracked. To do this, add a tagScreen call in the viewDidAppear call for each of your views. If your project doesn’t define this function, create it from the definition below. This should live in the view controller code. It is not recommended to append the name ‘screen’ to each screen as this is redundant when viewing the data online.

```objective-c
- (void)viewDidAppear:(BOOL)animated
{
	[[LocalyticsSession sharedLocalyticsSession] tagScreen:@"Good Name For This Screen."]; // examples: splash, game, options
}
```

## Sample Application Skeleton
To see it all at once, here is the skeleton of an instrumented iPhone application. Shown here is the Application’s AppDelegate.m file, because no other files need to be touched in the instrumentation process. (Although it is possible to add the event Tagging code to any file).

```objective-c
SampleAppDelegate.m
#import "Localytics_TestAppDelegate.h"
#import "Localytics_TestViewController.h"
 
// Pre defined event text.
#define EVENT_RESET_BUTTON @"RESET_BUTTON"
 
@implementation Localytics_TestAppDelegate
 
@synthesize window, viewController;
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	[window addSubview:viewController.view];
	[window makeKeyAndVisible];
 
	// Open Localytics Session
	[[LocalyticsSession sharedLocalyticsSession] startSession:MY_APP_KEY];
}
 
- (void)applicationWillEnterForeground:(UIApplication *)application {
	// Attempt to resume the existing session or create a new one.
	[[LocalyticsSession sharedLocalyticsSession] resume];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
 
- (void)applicationDidEnterBackground:(UIApplication *)application {
	// close the session before entering the background
	[[LocalyticsSession sharedLocalyticsSession] close];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
 
- (void)applicationWillTerminate:(UIApplication *)application {
	// Close Localytics Session in the case where the OS terminates the app
	[[LocalyticsSession sharedLocalyticsSession] close];
	[[LocalyticsSession sharedLocalyticsSession] upload];
}
 
// An action tied to a button which is defined in the view controller.
// Shown here for an example of when to use tags
- (void)restButtonClicked(id)sender {
	[[LocalyticsSession sharedLocalyticsSession] tagEvent:EVENT_RESET_BUTTON];
}
 
- (void)dealloc {
	[viewController release];
	[window release];
	[super dealloc];
}
@end
```

## Debugging No Data/Troubleshooting
Here are some common steps for debugging instrumentation which does not appear in the UI:
1. Double check the app key. Is it missing a first and last character or does it have an extra space somewhere?
2. Enable tracing. Debugging traces can be turned on by setting DO_LOCALYTICS_LOGGING (top of LocalyticsSession.h) to true. This will show whether the session was successfully opened, and whether the upload returned a 202 as expected or something else.
3. If you are still having trouble contact support@localytics.com and provide the logs from step 2.
If you see the error: /LocalyticsSession.m:830: error: request for member ‘__forwarding’ in something not a structure or union change your compiler to be Apple LLVM compiler 3.1 or above.

## Deprecated Libraries
Support for all libraries before version 2.0 has been deprecated and all customers are encouraged to upgrade to a more recent version of the libraries. New features may not be supported and and ongoing support for event and session tracking will be gradually decreased.
