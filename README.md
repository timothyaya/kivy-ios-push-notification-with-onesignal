# kivy-ios-push-notification-with-onesignal

# OneSignal iOS SDK Setup with Kivy-iOS

This guide covers integrating [OneSignal](https://onesignal.com) push notifications with a Kivy-iOS project.

---

## 1. Setup OneSignal SDK

### 📘 Based on: [OneSignal iOS SDK Setup Guide](https://documentation.onesignal.com/docs/ios-sdk-setup)

### 1.1 Select Xcode version

Use the Xcode version used to create your project:

```bash
sudo xcode-select -s /Applications/Xcode15.app
```
1.2 Enable Push and Background Capabilities
Go to App Target > Signing & Capabilities:

Enable Push Notifications

Enable Background Modes → Check Remote notifications

1.3 Setup App Group
In the same section:

Add App Groups

Add container: group.bundleid.onesignal

1.4 Create Notification Service Extension
Go to File > New > Target > Notification Service Extension

Set Product Name to: OneSignalNotificationServiceExtension

Ensure Project matches your app's name

Do not activate the new scheme when prompted

1.5 Configure Notification Service Target:
In the newly created OneSignalNotificationServiceExtension target:

In General, set Minimum Deployment to match your main target

Under Signing & Capabilities, add the same App Group container used in the main app

1.6 Replace NotificationService.swift Code:
Replace the default Swift file with the following:

```
import UserNotifications
import OneSignalExtension

class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var receivedRequest: UNNotificationRequest!
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest, withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.receivedRequest = request
        self.contentHandler = contentHandler
        self.bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        if let bestAttemptContent = bestAttemptContent {
            // Uncomment to debug
            // print("Running NotificationServiceExtension")
            // bestAttemptContent.body = "[Modified] " + bestAttemptContent.body

            OneSignalExtension.didReceiveNotificationExtensionRequest(
                self.receivedRequest,
                with: bestAttemptContent,
                withContentHandler: self.contentHandler
            )
        }
    }

    override func serviceExtensionTimeWillExpire() {
        if let contentHandler = contentHandler, let bestAttemptContent = bestAttemptContent {
            OneSignalExtension.serviceExtensionTimeWillExpireRequest(
                self.receivedRequest,
                with: self.bestAttemptContent
            )
            contentHandler(bestAttemptContent)
        }
    }
}
```
2. CocoaPods Setup
   tried to use "add package dependencies", not working:
   - onesignal 5.2.11 download fetal
   - specified 5.2.10, OneSignalInAppMessages' (no such file) <--got this error
📘 Based on: Setup Instructions (Google Docs) (https://docs.google.com/document/d/1fBYmxJu7Qm-ab8hD4aSME7M6rqYmnMQFfUbMFgtg9mQ/edit?tab=t.0)
2.1 Install CocoaPods
```
sudo gem install cocoapods
sudo gem install -n /usr/local/bin cocoapods
pod setup
```
2.2 Initialize Podfile
Navigate to the iOS project directory:
```
cd app-ios
pod init
open -a Xcode Podfile
```

Edit your Podfile as follows:
```
platform :ios, '15.0'

target 'mainapp' do
  use_frameworks! :linkage => :static
  pod 'OneSignal/OneSignal', '>= 5.2.9', '< 6.0'
  pod 'OneSignal/OneSignalInAppMessages', '>= 5.2.9', '< 6.0'
  pod 'OneSignal/OneSignalLocation', '>= 5.2.9', '< 6.0'
end

target 'OneSignalNotificationServiceExtension' do
  use_frameworks! :linkage => :static
  pod 'OneSignalXCFramework', '>= 5.2.9', '< 6.0'
end
```
2.3 Install Pods
```
pod install
```
2.4 Open Workspace
Open .xcworkspace (not .xcodeproj):

2.5 Build Settings：

If this step is not done, an error saying 'onesignalId' not found will occur later when using NSString* userid = OneSignal.User.onesignalId;.

Update build settings:

Enable Modules (C and Objective-C): Yes

Other Linker Flags: add -ObjC and $(inherited)

Framework Search Paths: add $(inherited)

Header Search Paths: add $(inherited)

In Pods > Build Settings:

Build Active Architecture Only → Release: No

Close Xcode and run:
```
pod install
```
3. Create .p8 APNs Key
Go to Apple Developer Center → Certificates, Identifiers & Profiles

Under Keys, create a new Apple Push Notification service (APNs) key

Under Identifiers, create a new identifier using your Bundle ID and enable Push Notifications

Save your Team ID

4. OneSignal Dashboard
In OneSignal dashboard:

Go to Settings > Apple Push

Upload .p8 key, Team ID, and Bundle ID

You will receive your OneSignal App ID

5. Kivy iOS App Integration
5.1 In main.py
Add inside on_build():
```
        if platform == 'ios':
            // not needed, bebacuse it's initialized in main.m
            // from pyobjus import autoclass, objc_str
            // self.OneSignal = autoclass('OneSignal')
            // mock_launch_options = objc_dict({})
            // appid = 'YOUR_APP_IP'
            // launch_options = objc_dict({})
            // self.OneSignal.setAppId_(appid)
            // self.OneSignal.initialize_withLaunchOptions_(appid,launch_options)

            def read_push_id():
                file_path = os.path.expanduser("~/Documents/pushid.txt")
                if os.path.exists(file_path):
                    with open(file_path, "r") as f:
                        return f.read().strip()
                return None
            pushid = read_push_id()
            print(f'pushid in python :{pushid}')
```
5.2 In main.m
Modify main() as follows:
import dependencies:
```
#import <OneSignalFramework/OneSignalFramework.h>
```

```
int main(int argc, char *argv[]) {
    int ret = 0;

    NSAutoreleasePool * pool = [[NSAutoreleasePool alloc] init];

   [OneSignal.Debug setLogLevel:ONE_S_LL_VERBOSE];
    // Initialize with your OneSignal App ID
    [OneSignal initialize:@"YOUR_APP_ID" withLaunchOptions:nil];
    // Use this method to prompt for push notifications.
    // We recommend removing this method after testing and instead use In-App Messages to prompt for notification permission.
    [OneSignal.Notifications requestPermission:^(BOOL accepted) {
      NSLog(@"User accepted notifications: %d", accepted);
        // get onesignal id
        NSString* userid = OneSignal.User.onesignalId;
        NSLog(@"User ID: %@", userid);

        //get subscription id <- this one for api send push notification
        NSString* pushid = OneSignal.User.pushSubscription.id;
        NSLog(@"push ID: %@", pushid);

        // sSave the file so that it can be accessed by the Python code.
        NSString *filePath = [NSHomeDirectory() stringByAppendingPathComponent:@"Documents/pushid.txt"];
        [pushid writeToFile:filePath atomically:YES encoding:NSUTF8StringEncoding error:nil];
    } fallbackToSettings:false];
```
notice: remember to change the YOUR_APP_IP to your app_id which retrived from onesignal website

7. Testing
Go to OneSignal dashboard:

Navigate to Messages > Push > New Push

Create and send a test notification

You’ve successfully set up OneSignal with a Kivy-iOS app!

8. test push notification:
   create api_key of onesignal : onesignal website -> click on Settings in the navigation menu on the left -> keys & IDs -> add key
   get onesignal id and subscription id from audience >> subscriptions in the navigation menu on the left
   test with python:
```
import requests

ONESIGNAL_APP_ID = "your_onesignal app id"
ONESIGNAL_API_KEY = "your api key"
TARGET_PLAYER_ID = "your subscription id"

url = "https://onesignal.com/api/v1/notifications"
headers = {
    "Content-Type": "application/json; charset=utf-8",
    "Authorization": f"Basic {ONESIGNAL_API_KEY}"
}

payload = {
    "app_id": ONESIGNAL_APP_ID,
    "include_player_ids": [TARGET_PLAYER_ID],  # 或改成 include_external_user_ids
    "headings": {"en": "test push notification"},
    "contents": {"en": "this is a test from python"},
    "priority": 10,
}

# send notification
response = requests.post(url, headers=headers, json=payload)

# print result
print("Status Code:", response.status_code)
print("Response Body:", response.text)
```

9. The next step is to place the api_key and app_id in a Google Cloud Function, so that when the API sends a push request, the function is triggered to automatically send the push notification.
   
