# RxRealm

[![Carthage Compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Version](https://img.shields.io/cocoapods/v/RxRealm.svg?style=flat)](http://cocoapods.org/pods/RxRealm)
[![License](https://img.shields.io/cocoapods/l/RxRealm.svg?style=flat)](http://cocoapods.org/pods/RxRealm)
[![Platform](https://img.shields.io/cocoapods/p/RxRealm.svg?style=flat)](http://cocoapods.org/pods/RxRealm)

This library is a thin wrapper around __RealmSwift__ ( [Realm Docs](https://realm.io/docs/swift/latest/) ).

**Table of contents:**

 1. Observing object collections
 2. Observing a single object
 3. Write transactions
 4. Automatically binding table and collection views
 5. Example app

## Observing object collections

RxRealm can be used to create `Observable`s from objects of type `Results`, `List`, `LinkingObjects` or `AnyRealmCollection`. These types are typically used to load and observe object collections from the Realm Mobile Database.

##### `Observable.collection(from:synchronizedStart:)`
Emits an event each time the collection changes:

```swift
let realm = try! Realm()
let laps = realm.objects(Lap.self)

Observable.collection(from: laps)
  .map { 
    laps in "\(laps.count) laps"
  }
  .subscribe(onNext: { text  in
    print(text)
  })
```

The above prints out "X laps" each time a lap is added or removed from the database. If you set `synchronizedStart` to `true` (the default value), the first element will be emitted synchronously - e.g. when you're binding UI you might not be able for an asynchronous notification to come through.

##### `Observable.array(from:synchronizedStart:)`
Upon each change fetches a snapshot of the Realm collection and converts it to an array value (for example if you want to use array methods on the collection):

```swift
let realm = try! Realm()
let laps = realm.objects(Lap.self)

Observable.array(from: laps)
  .map { array in
    return array.prefix(3) //slice of first 3 items
  }
  .subscribe(onNext: { text  in
    print(text)
  })
```

##### `Observable.changeset(from:synchronizedStart:)`
Emits every time the collection changes and provides the exact indexes that has been deleted, inserted or updated:

```swift
let realm = try! Realm()
let laps = realm.objects(Lap.self))

Observable.changeset(from: laps)
  .subscribe(onNext: { results, changes in
    if let changes = changes {
      // it's an update
      print(results)
      print("deleted: \(changes.deleted)")
      print("inserted: \(changes.inserted)")
      print("updated: \(changes.updated)")
    } else {
      // it's the initial data
      print(results)
    }
  })
```

##### `Observable.arrayWithChangeset(from:synchronizedStart:)`
Combines the result of `Observable.array(from:)` and `Observable.changeset(from:)` returning an `Observable<Array<T>, RealmChangeset?>`

```swift
let realm = try! Realm()
let laps = realm.objects(Lap.self))

Observable.arrayWithChangeset(from: laps)
  .subscribe(onNext: { array, changes in
    if let changes = changes {
    // it's an update
    print(array.first)
    print("deleted: \(changes.deleted)")
    print("inserted: \(changes.inserted)")
    print("updated: \(changes.updated)")
  } else {
    // it's the initial data
    print(array)
  }
  })
```

## Observing a single object

There's a separate API to make it easier to observe a single object (it creates `Results` behind the scenes):

```swift
Observable.from(object: ticker)
    .map { ticker -> String in
        return "\(ticker.ticks) ticks"
    }
    .bindTo(footer.rx.text)
```

This API uses the primary key of the object to query the database for it and observe for change notifications. Observing objects without a primary key does not work.

## Write transactions

##### `rx.add()`

Writing objects to **existing** realm reference. You can add newly created objects to a Realm that you already have initialized:

```swift
let realm = try! Realm()
let messages = [Message("hello"), Message("world")]

Observable.from(messages)
  .subscribe(realm.rx.add())
```

Be careful, this will retain your Realm until the `Observable` completes or errors out.

##### `Realm.rx.add()`

Writing to the default Realm. You can leave it to RxRealm to grab the default Realm on any thread your subscribe and write objects to it:

```swift
let messages = [Message("hello"), Message("world")]

Observable.from(messages)
  .subscribe(Realm.rx.add())
```

###### `Realm.rx.add(configuration:)`

Writing to a **custom** Realm. If you want to switch threads and not use the default Realm, provide a `Realm.Configuration`:

```swift
var config = Realm.Configuration()
/* custom configuration settings */

let messages = [Message("hello"), Message("world")]
Observable.from(messages)
  .observeOn( /* you can switch threads here */ )     
  .subscribe(Realm.rx.add(configuration: config))
```

If you want to create a Realm on a different thread manually, allowing you to handle errors, you can do that too:

```swift
let messages = [Message("hello"), Message("world")]

Observable.from(messages)
  .observeOn( /* you can switch threads here */ )
  .subscribe(onNext: {messages in
    let realm = try! Realm()
    try! realm.write {
      realm.add(messages)
    }
  })
```

##### `rx.delete()`

Deleting object(s) from an existing realm reference:

```swift
let realm = try! Realm()
let messages = realm.objects(Message.self)
Observable.from(messages)
  .subscribe(realm.rx.delete())
```

Be careful, this will retain your realm until the `Observable` completes or errors out.

##### `Realm.rx.delete()`

Deleting from the object's realm automatically. You can leave it to RxRealm to grab the Realm from the first object and use it:

```swift
Observable.from(someCollectionOfPersistedObjects)
  .subscribe(Realm.rx.delete())
```

## Automatically binding table and collection views

RxRealm does not depend on UIKit/Cocoa and it doesn't provide built-in way to bind Realm collections to UI components.

#### a) Non-animated binding

You can use the built-in RxCocoa `bindTo(_:)` method, which will automatically drive your table view from your Realm results:

```swift
Observable.changeset(from: [Realm collection] )
  .bindTo(tableView.rx.realmChanges(dataSource))
  .addDisposableTo(bag)
```

#### b) Animated binding with RxRealmDataSources

There is a separate library __`RxRealmDataSources`__ [link](https://github.com/RxSwiftCommunity/RxRealmDataSources), which mimics the default data sources library behavior for RxSwift.

`RxRealmDataSources` allows you to bind directly an observable collection of Realm objects to a table or collection view. Here's how the code to bind a collection of laps to a table view looks like:

```swift
// create data source
let dataSource = RxTableViewRealmDataSource<Lap>(
  cellIdentifier: "Cell", cellType: PersonCell.self) {cell, ip, lap in
    cell.customLabel.text = "\(ip.row). \(lap.text)"
}

// RxRealm to get Observable<Results>
let realm = try! Realm()
let lapsList = realm.objects(Timer.self).first!.laps
let laps = Observable.changeset(from: lapsList)

// bind to table view
laps
  .bindTo(tableView.rx.realmChanges(dataSource))
  .addDisposableTo(bag)
```

The data source will reflect all changes via animations to the table view:

![RxRealm animated changes](assets/animatedChanges.gif)

If you want to learn more about the features beyond animating changes, check the __`RxRealmDataSources`__ [README](https://github.com/RxSwiftCommunity/RxRealmDataSources).

## Example app

To run the example project, clone the repo, and run `pod install` from the Example directory first. The app uses RxSwift, RxCocoa using RealmSwift, RxRealm to observe Results from Realm.

Further you're welcome to peak into the __RxRealmTests__ folder of the example app, which features the library's unit tests.

## Installation

This library depends on both __RxSwift__ and __RealmSwift__ 1.0+.

#### CocoaPods

RxRealm requires CocoaPods 1.1.x or higher.

RxRealm is available through [CocoaPods](http://cocoapods.org). To install it, simply add the following line to your Podfile:

```ruby
pod "RxRealm"
```

#### Carthage

To integrate RxRealm into your Xcode project using Carthage, specify it in your `Cartfile`:

```ogdl
github "RxSwiftCommunity/RxRealm"
```

Run `carthage update` to build the framework and drag the built `RxRealm.framework` into your Xcode project.

## TODO

* Test add platforms and add compatibility for the pod

## License

This library belongs to _RxSwiftCommunity_. Maintainer is [Marin Todorov](https://github.com/icanzilb).

RxRealm is available under the MIT license. See the LICENSE file for more info.
