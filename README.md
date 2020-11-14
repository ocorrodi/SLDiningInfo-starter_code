---
title: Introduction to iOS Development
layout: page
---

## iOS Basics

When iOS 2 first allowed developers to add third party apps to be added to the
app store, it completely changed the way people interacted with their phones.
While apps seem commonplace these days, they remain one of the most effective
and powerful ways to interact with users, and iOS is the best platform to build
them for (no really).

## Application Structure

### Model-View-Controller (MVC)

MVC is the primary design pattern employed for iOS development. It divides the
application into three parts:

  1. Model - The actual data and information stored in the app. Could be Tasks,
     Game State, Restaurant Ratings, etc. These classes may have no superclass,
     or simply a superclass of NSObject. If they need to be persistent, they
     usually are part of CoreData.
  2. View - What the user sees. View objects are usually subclasses of UIView,
     and should _only_ have methods that modify visual appearance. Try to use
     Storyboard to edit your views.
  3. Controller - This is where all of the logic happens. The controller reads
     and writes data to the model, performs logical tasks, and updates the Views
     when necessary. Controllers are usually subclasses of UIViewController, and
     are where you will do the meat of your coding.

### Delegation

Another design pattern common in iOS is delegation. One common use for
delegation is for View elements, which are responsible for recieving user input,
want to tell a Controller that their state has changed. This helps keep tasks
separate, as suggested by MVC, by allowing the Controller to manage computation,
even when it is based off a view event.

`Delegate` is a commone postfix on `procotol` objects. We will examine protocols
later, but essentially this means that a controller must provide a specific
interface in order to act as a view delegate.

### Threading

Whenever you are developing a user interface, especially one on
a resource-constrained platform, you should focus on making the main thread
(a.k.a. UI thread) free from work. It should be used exclusively for updating
user interface elements, to minimize lag.

If you are reading a file or making a network request, you can do work
asynchronously on a background thread with the following code:

```swift
DispatchQueue.global(qos: .background).async {
    // CODE GOES HERE
}
```

Frequently, we want to call back to the main thread once we have finished our
work, we can do that like so:

```swift
DispatchQueue.global(qos: .background).async {
    let result = longNetworkingOperation()
    DispatchQueue.main.async {
        updateUI(result)
    }
}
```

### UIKit

UIKit is Apple's developer library for iOS technology. _Always_ use it where
possible. One great thing about developing for iOS is the number of reliable
tools and APIs that Apple supports. Apple generally supports and improves them
well (except for Apple Watch), so really try not to reinvent the wheel.

The classes mentioned above, UIView and UIViewController, are the backbone of
iOS apps. Those classes (and their subclasses) will make up the majority of your
application.

### App Delegate

The _app delegate_ is the central class that controls the app on startup and
shutdown, and handle external events like push notifications. For our tutorial,
we won't modify it much, but if you do anything with third party libraries, you
usually need to configure that here.

### Storyboard

Storyboard is Xcode's built-in layout builder. It is versatile, and will connect
directly to your code via annotations. It's a visual tool, so you really have to
see it in action to understand what it does.

### View Controller

As I mentioned, view controllers are extremely popular items to subclass. There
are three methods to highlight, they all correspond to the life cycle of the
view:

  * `viewDidLoad`: Do all initial configuration, like background color and
     initial values for text labels, here. This is called only once.
  * `viewDidAppear`: Do all things that must occur _every time the view appears
     here_. It is best to use `viewDidLoad` if you can, but if some UI or
     animation needs updating whenever the user sees that, do it here.
  * `viewDidDisappear`: This method is useful for stopping animations or UI
     processes that are just wasting resources when the user isn't looking at
     the view.

__Always__ call the `super` method before doing your own implementation.

# Today's App

We are going to make a simple application that fetchs information via a [RESTful
API](https://en.wikipedia.org/wiki/Representational_state_transfer), and
displays it on your phone. You'll find that working with a RESTful API is
a valuable skill, especially when building iOS applications for hackathons and
other small projects.

For this example, we will be using the [ScottyLabs Dining
API](https://scottylabs.org/dining-api/), which we can use to get information
about the information and hours for Carnegie Mellon Dining facilities.

## Setting Up Your Project

Configuration steps:

  1. Open Xcode
  2. Create New XCode Project
  3. Single View Application [next]
  4. Product Name: SLDiningInfo
  5. Organization Name: Your Name
  6. Organization Identifier: edu.cmu.andrewid
  7. Language Swift
  8. Devices Universal
  9. Leave both the core data and testing boxes unchecked (you should write
     tests, but we want today's talk to be brief)

Now **open your terminal**. (Ha! You probably thought iOS development wasn't
going to involve a terminal, but alas, it does.)

## Importing Libraries

[SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON) and
[Alamofire](https://github.com/Alamofire/Alamofire) are two extremely popular
Swift libraries that make working with a JSON RESTful API a breeze.

We will use [CocoaPods](https://github.com/Carthage/Carthage), a package manager
for iOS and macOS development, to import them.

To do this, we need to create a new `Podfile` file (which acts as
a package manifest). Then create allow the package manager to re-arrange our
project into a workspace (which is why XCode should be closed).

Navigate to your project directory and run the following commands, some of which
may take a long time:

```bash
# Execute this first line by itself, it takes a while
sudo gem install cocoapods
# Copy all the way down to EOT here, this will have freaky output, but don't
# worry, it works
>/dev/null cat <<EOT >> Podfile
source 'https://github.com/CocoaPods/Specs.git'
platform :ios, '10.0'
use_frameworks!
target 'SLDiningInfo' do
    pod 'SwiftyJSON'
    pod 'Alamofire', '~> 4.3'
    end
EOT
# Satisfy Framework dependencies
pod repo update && pod install
# Finally execute this command to open XCode again
open SLDiningInfo.xcworkspace
```

Finally, we can start writing real code!

## Defining a Model

Create a new swift file (`CMD+N`) named `DiningLocation.swift`. Ensure that the
`SLDiningInfo` group with the yellow folder is selected in the combo-box, and
that **only** the `SLDiningInfo` target is selected.

For now, our `struct` will have three fields, `name`, `description` and
`location`, all of them being strings.

We are using a `struct` because we are creating immutable value types, and there
is no need for referential access to them, but this decision might not be the
same if you want to do more than simply display information.

The structure can be defined like this:

```swift
// DiningLocation.swift
struct DiningLocation {
    let name: String
    let description: String
    let location: String
}
```

### Creating with JSON

The dining API returns JSON responses to requests. We will use a static factory
method to create `DiningLocation` objects from the returned JSON. Arbitrary JSON
will not always have the right fields; we could write a validator, but instead
our factory will have an optional return type.

We will add constants to hold the key descriptions in `DiningAPI.swift` which
will manage all of our API information. Keeping our constants in this file
allows us to migrate API versions most simply.

```swift
// DiningAPI.swift
class DiningAPI {
    static let queryUrl = "https://apis.scottylabs.org/dining/v1/locations"

    // Constants for accessing JSON properties
    // Compliant with ScottyLabs Dining API v1
    struct Keys {
        static let locationList = "locations"
        struct Location {
            static let name = "name";
            static let description = "description";
            static let location = "location";
        }
    }
}
```


JSON data. So far I haven't found a better way to do this:

```swift
// DiningLocation.swift
import SwiftyJSON

struct DiningLocation {
    let name: String
    let description: String
    let location: String

    static func fromJson(json: JSON) -> DiningLocation? {
        if let tmpName = json[DiningAPI.Keys.Location.name].string,
            let tmpDesc = json[DiningAPI.Keys.Location.description].string,
            let tmpLoc = json[DiningAPI.Keys.Location.location].string {
            return DiningLocation(name: tmpName, description: tmpDesc, location: tmpLoc)
        }
        return nil
    }
}
```

## Retrieving JSON Data

The following code describes how you can retrieve JSON data from a RESTful API.

Since this is a short project, the focus is on simplicity of code and
correctness, not on fine-grained error handing. Therefore, any error is only
seen as an empty list by the caller.

For those of you unfamiliar with functional programming paradigms (many of which
are stolen by Swift), the `map` function is taking an array of JSON objects, and
calling our `DiningLocation` static factory on each JSON object. Since our
static factory produces `nil` if JSON cannot be parsed, we use the `filter`
function to remove any `nil` entries from our array.

```swift
// DiningAPI.swift
import SwiftyJSON
import Alamofire
class DiningAPI {
    static let queryUrl = "https://apis.scottylabs.org/dining/v1/locations"

    /**
    Constants for accessing JSON properties. Compliant with ScottyLabs Dining API v1
    */
    struct Keys {
        static let locationList = "locations"
        struct Location {
            static let name = "name"
            static let description = "description"
            static let location = "location"
            static let keywords = "keywords"
            static let times = "times"
            struct Times {
                static let start = "start"
                static let end = "start"
                struct TimeUnit {
                    static let day = "day"
                    static let hour = "hour"
                    static let minute = "minute"
                }
            }
        }
    }

    /**
    Asynchronously fetches dinining location information from the ScottyLabs dining API

    When successful, the completion handler is called with a array of `DiningLocation` objects,
    otherwise, and empty array is returned.

     :param: onComplete Completion handler to deal with retrieved information,
     called on main thread.
    */
    static func fetchLocations(complete onComplete: @escaping ([DiningLocation]) -> ()) {
        Alamofire.request(queryUrl)
          .responseJSON { response in
            // Check if there was an error with the request
            guard response.result.error == nil else {
                print("Error: \(response.result.error!)")
                onComplete([DiningLocation]())
                return
            }
            // Here we use Grand Central Dispatch (GCD) to process our JSON on a background queue.
            // We should still call the completion handler on the main queue though
            DispatchQueue.global(qos: .background).async {
                // Check if the array can be properly parsed
                guard let jsonArr = SwiftyJSON.JSON(response.value!)["locations"].array else {
                    print("Error: Result recieved, but unparceable")
                    // Call the completion handler on the main thread with an empty array to signify
                    // that no results were found
                    DispatchQueue.main.async {
                        onComplete([DiningLocation]())
                    }
                    return
                }
                // Create all possible dining locations, removing those whose JSON cannot be parsed
                // We can safely force the non-optional cast because the filter removes all `nil`
                // array elements
                let diningLocations = jsonArr.map({ DiningLocation.fromJson(json: $0) })
                    .filter( { $0 != nil })
                DispatchQueue.main.async {
                    onComplete(diningLocations as! [DiningLocation])
                }
            }
        }
    }
}

```

## Subclassing UITableViewController

We will subclass `UITableViewController`, because it is very applicable to a lot
of different applications, and requires way less work than `UIScrollView`.

### Adding Infomation

We need to set up our `ViewController` to be a subclass of
`UITableViewController`. Change the class declaration to the following:

```swift
// ViewController.swift
class ViewController: UITableViewController {
```

Xcode probably added method skeletons to your class, delete the one that says
`didRecieveMemoryWarning`, but keep the `viewDidLoad`.

Okay, now lets add our array of objects that will be the underlying model for
the table. Let's add a field called `locationList` that will hold our location
information. It will be initialized to an empty list, but we will also add
a function to fetch new data and update the list.

```swift
// ViewController.swift
var locationList = [DiningLocation]()

private func refreshLocationList() {
    DiningAPI.fetchLocations {
        (newLocationList) in
        // Update our underlying array
        self.locationList = newLocationList
        // Request that the table update to reflect the new information
        self.tableView.reloadData()
    }
}
```

In `viewDidLoad`, we will call our refresh function to retrieve our information.
We DO NOT override the constructor. It is not idiomatic in iOS programming to
override `ViewController` constructors, in which interface builder outlets may
not be initialized yet. Use `viewDidLoad` for all initialization, and
`viewDidAppear` for animations or other things that should only occur after the
user is viewing your screen.

```swift
// ViewController.swift
override func viewWillAppear() {
    super.viewDidLoad()
    // Fetch an initial list of results
    refreshLocationList()
}
```

Now that we have information, we need to configure the table to show it.

### Table Size

These methods correspond to protocols called `UITableViewDelegate` and
`UITableViewDataSource`, which are implemented by `UITableViewController`.

First, we need to tell the table how many rows to display:

```swift
// ViewController.swift
override func numberOfSections(in tableView: UITableView) -> Int {
    return 1
}

override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return locationList.count
}
```

### Populate Table

Next, for a given row in the table, we have to tell the table what information to show:

```swift
// ViewController.swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    var cell = tableView.dequeueReusableCell(withIdentifier: "LocationCell")
    if cell == nil {
        cell = UITableViewCell(style: UITableViewCellStyle.value1, reuseIdentifier: "LocationCell")
    }

    let snapshot = locationList
    if let cell = cell, indexPath.row < snapshot.count {
        // These are the only line that actually set up the cell
        let location = snapshot[indexPath.row]
        cell.textLabel?.text = location.name
        cell.detailTextLabel?.text = location.location
    }

    return cell!
}
```

The excessive `if let` is because `dequeue` is a notoriously messy function, so
this is essentially a bunch of `nil` checking. The `dequeue` is actually
a beautiful memory management tool that allows for large scrolling tables, but
it is tough to deal with, so just copy paste this code when you need to, and
change the part that modifies the cell.

## Designing the Interface

First add a "Label" item to the view, and run your app, just to see the simulator working.

Now for our interface:

  1. Select `Main.storyboard` from the file browser
  2. Click on the white bar that says view controller. Delete it. Drag
     a `UITableViewController` into the storyboard area from the object browser
     in the bottom right. The icon looks like a yellow circle with lined paper
     on it. In its Identity Inspector (Newspaper icon), set the class to
     `ViewController` which is our custom class the project created by default.
  3. In the menu bar, go to `Editor->Embed in..->Navigation Controller`
  4. Click on the Navigation bar on the table VC (View Controller). In the
     attributes inspector on the right (it looks like a line with a downward
     arrow), enter a title like "Location Information".
  5. In the outline on the left, select the "Table View Cell", or click on the
     cell in the image. In the attributes inspector where it says "Style",
     select `Subtitle`. Set it's identifier to `LocationCell`.
  6. Select the Navigation Controller and check the checkbox that says "Is
     Initial View Controller" under the tab whose icon looks roughly like
     `-\/-`.

__Hit run!!!__

Tutorial taken from https://github.com/ScottyLabs/IntroToSwift/blob/gh-pages/ios.md for CrashCourse 2020 presented by ScottyLabs.
