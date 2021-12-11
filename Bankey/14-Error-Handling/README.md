# Error Handling

![](images/3a.png)

It's easy to stick to happy path scenarios when building apps. But just as important is adding affordances for when things go wrong.

Let's look at a couple of ways we can anticipate things going wrong and then adding affordance to handle them.

## Handling failed network calls

### Failed profile fetch

First, if we haven't already, let's comment back in our `fetchData` call.

**AccountSummaryViewController**

```swift
// MARK: - Setup
extension AccountSummaryViewController {
    private func setup() {
        setupNavigationBar()
        ...
        fetchData() // 
    }
```

Then let's force our `fetchProfile` network call to fail by commenting out every line of code except this one here.

**AccountSummaryViewController+Networking**

```swift
completion(.failure(.decodingError))
```

This is an easy way to test and force and error in your app. Just hard code it and make it happen.

When we run the app now our screen just sits there stuck loading skeletons.

<img src="images/0.png" width="300">

Let's pop up an alert and give them at left some feedback that we know something is going wrong.

**AccountSummaryViewController** 

```swift

private func configureTableCells(with accounts: [Account]) {
}

private func showErrorAlert() {
    let alert = UIAlertController(title: "Network Error",
                                  message: "Please check your network connectivity and try again.",
                                  preferredStyle: .alert)
    
    alert.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
    
    present(alert, animated: true, completion: nil)
}
```

And let's call it from where our fetch profile fails.

```swift
fetchProfile(forUserId: userId) { result in
    switch result {
    case .success(let profile):
        self.profile = profile
    case .failure(let error):
        self.showErrorAlert() //
    }
    group.leave()
}
```

If we run the app now we see an error alert pop-up.

<img src="images/1.png" width="300">

### Challenge

OK - that's not bad. But what if we want to display a different error message based on the type of error that occurred?

Right now we can have one of two errors:

```swift
enum NetworkError: Error {
    case serverError
    case decodingError
}
```

Why don't you see if you can detect what kind of error gets by adding a `switch` statement here:

```swift
fetchProfile(forUserId: userId) { result in
    switch result {
    case .success(let profile):
        self.profile = profile
    case .failure(let error):
        // 🕹 Game on switch here...
        self.showErrorAlert() //
    }
    group.leave()
}
```

And then based on the error return, refactor the `showErrorAlert` func to take a `title` and `message` String and display one of the following messages:

case serverError:

 - `title` = `Server Error`
 - `message` = `Ensure you are connected to the internet. Please try again.`

case decodingError:

 - `title` = `Decoding Error`
 - `message` = `We could not process your request. Please try again.`


Give that a go. Comeback and we'll do it together. 

### Solution

Alright, let's refactor the alert funct first to take a title and a message.

```swift
private func showErrorAlert(title: String, message: String) {
    let alert = UIAlertController(title: title,
                                  message: message,
                                  preferredStyle: .alert)
    
    alert.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
    
    present(alert, animated: true, completion: nil)
}
```

Then let's add the error switch and let it determine what the `title` and `message` should be before passing it show alert.

```swift
case .failure(let error):
    let title: String
    let message: String
    switch error {
    case .serverError:
        title = "Server Error"
        message = "Ensure you are connected to the internet. Please try again."
    case .decodingError:
        title = "Decoding Error"
        message = "We could not process your request. Please try again."
    }
    self.showErrorAlert(title: title, message: message)
}
```

OK this works.

- Demo network error.
- Demo server error.

Discussion

- talk about code readability and how to refactor code so the abstractions are all at the same level

Let's extract a method to make it read like this.

```swift
fetchProfile(forUserId: userId) { result in
    switch result {
    case .success(let profile):
        self.profile = profile
    case .failure(let error):
        self.displayError(error)
    }
    group.leave()
}
```

And

```swift
private func displayError(_ error: NetworkError) {
    let title: String
    let message: String
    switch error {
    case .serverError:
        title = "Server Error"
        message = "We could not process your request. Please try again."
    case .decodingError:
        title = "Network Error"
        message = "Ensure you are connected to the internet. Please try again."
    }
    self.showErrorAlert(title: title, message: message)
}
```

Now everything reads more nicely and we don't get lost in the details of implementation.

Let's do the same for `fetchAccount`.

```swift
fetchAccounts(forUserId: userId) { result in
    switch result {
    case .success(let accounts):
        self.accounts = accounts
    case .failure(let error):
        self.displayError(error) // 
    }
    group.leave()
}
```

And let's comment back in our `fetchProfile` network code.

```swift
extension AccountSummaryViewController {
    func fetchProfile(forUserId userId: String, completion: @escaping (Result<Profile,NetworkError>) -> Void) {
        let url = URL(string: "https://fierce-retreat-36855.herokuapp.com/bankey/profile/\(userId)")!

        URLSession.shared.dataTask(with: url) { data, response, error in
            DispatchQueue.main.async {
                guard let data = data, error == nil else {
                    completion(.failure(.serverError))
                    return
                }

                do {
                    let profile = try JSONDecoder().decode(Profile.self, from: data)
                    completion(.success(profile))
                } catch {
                    completion(.failure(.decodingError))
                }
            }
        }.resume()
    }
}
```

Good stuff. Network error cases handled.

### Save your work

```
> git add .
> git commit -m "feat: Handle network errors"
```

## Unit testing network failures

As good as manually testing network errors are, what's even better is if we can automate them. 

![](images/6.png)

The challenge is how do we fake the network? The network is a real thing.

- We can't control what it does
- We can't make it fail in ways that we want

On can we?

In this section we are going to look at a super powerful unit testing technique called *dependency injection* which is perfect for unit testing situations like this. And in this section you are going to learn how it works, and why it's so powerful for getting to those hard to reach places in our code.

### What is Dependency Injection

![](images/2.png)

- What is it?
- How does it work?
- Why is is so handy for unit testing?

### Define the protocol

The protocol is the thing we want to inject into our ViewController. Create a new section called `Networking` and create a new file in there called `ProfileManager`.

![](images/4a.png)

**ProfileManager**

```swift
protocol ProfileManageable: AnyObject {
    func fetchProfile(forUserId userId: String, completion: @escaping (Result<Profile,NetworkError>) -> Void)
}
```

Discussion:

- Why `AnyObject`

### Merge the protocol with the real implementation

Now we already have the real version of this protocol going in our app. In order to unit test it however, we need to merge our protocol with the real thing.

Currently this code is embedded in our `AccountSummaryViewController+Networking` extention. Let's extract it and all related code into it's this newly created file, and make it implement our protocol.

**ProfileManager**

```swift
protocol ProfileManageable: AnyObject {
    func fetchProfile(forUserId userId: String, completion: @escaping (Result<Profile,NetworkError>) -> Void)
}

enum NetworkError: Error {
    case serverError
    case decodingError
}

struct Profile: Codable {
    let id: String
    let firstName: String
    let lastName: String
    
    enum CodingKeys: String, CodingKey {
        case id
        case firstName = "first_name"
        case lastName = "last_name"
    }
}

class ProfileManager: ProfileManageable {
    func fetchProfile(forUserId userId: String, completion: @escaping (Result<Profile,NetworkError>) -> Void) {
        let url = URL(string: "https://fierce-retreat-36855.herokuapp.com/bankey/profile/\(userId)")!

        URLSession.shared.dataTask(with: url) { data, response, error in
            DispatchQueue.main.async {
                guard let data = data, error == nil else {
                    completion(.failure(.serverError))
                    return
                }

                do {
                    let profile = try JSONDecoder().decode(Profile.self, from: data)
                    completion(.success(profile))
                } catch {
                    completion(.failure(.decodingError))
                }
            }
        }.resume()
    }
}
```

Discussion:

- Note how we are implementing the protocol
- Note how this is a `class`.

Now we just need to update the `AccountSummaryViewController` to use it.

**AccountSummaryViewController**

```swift
// Components
...
let refreshControl = UIRefreshControl()
    
// Networking
var profileManageable: ProfileManageable = ProfileManager()

// MARK: - Networking
extension AccountSummaryViewController {
    private func fetchData() {
		...        
        group.enter()
        profileManager.fetchProfile(forUserId: userId) { //

```

Run the app. Everything should still work.

But this is magic 🌈. Now that we have this protocol defined, we can *inject* into our view controller whatever we want.

Let's now head over to the unit testing section of our app, and see what tests we can write there.

## Unit testing view controllers

Unit testing view controllers can be tricky.

- View hierarchies between tests and production don't always match
- Getting access to view and view controllers in tests can be tough
- And then you've got view controller life cycle stuff to worry about. Gotta know when `viewDidLoad` is called an how that affects your tests.

Fortunately, we don't have to test everything about the view controller. Only the bits that change or the things we care about. 

Let's look at a couple of techniques for making our view controllers more testable, and then automating those things we can about in our tests.

### Creating the unit test

First let's create a placeholder for all our account summary unit tests.

![](images/8.png)

And then into there copy the following code.

**AccountSummaryViewControllerTests**

```swift
import Foundation
import XCTest

@testable import Bankey

class ProfileNetworkingTests: XCTestCase {
    var vc: AccountSummaryViewController!
    
    override func setUp() {
        super.setUp()
        vc = AccountSummaryViewController()
        // vc.loadViewIfNeeded()
    }
    
    func testSomething() throws {
        
    }
}
```

Discussion

- `vc.loadViewIfNeeded()`

We are just going to leave this here for now, but will return to it when we find something we'd like to test.

But first let's talk a little more about unit testing.

### Looking for effects

Unit testing is about looking for effects. You do something - you expect something to change.

For us, we want to test that:

- when fetchProfile succeeds - profile gets set
- when fetchProfile fails - an alert pops up with one of two error messages

It all really comes down how do we unit test this code here?

**AccountSummaryViewController**

```swift
// MARK: - Networking
extension AccountSummaryViewController {
    private func displayError(_ error: NetworkError) {
        let title: String
        let message: String
        switch error {
        case .serverError:
            title = "Server Error"
            message = "We could not process your request. Please try again."
        case .decodingError:
            title = "Network Error"
            message = "Ensure you are connected to the internet. Please try again."
        }
        self.showErrorAlert(title: title, message: message)
    }
}
```

How can we ensure that when a `severError` occurs, one kind of alert gets displayed, yet when a decodingError` occurs another gets displayed.
     
### The power of extraction

![](images/7.png)

One of the most powerful things you can do while unit testing is extract logic. By extracting things out of your view controllers, and breaking them down into smaller testable bits, you suddenly gain the ability to take things that look big and daunting, and break them down into smaller more manageable peices.

Take this display function here. It's really doing two things.

```swift
private func displayError(_ error: NetworkError) {
    let title: String
    let message: String
    switch error {
    case .serverError:
        title = "Server Error"
        message = "We could not process your request. Please try again."
    case .decodingError:
        title = "Network Error"
        message = "Ensure you are connected to the internet. Please try again."
    }
    self.showErrorAlert(title: title, message: message)
}
```

1. It's determining what the `title` and `message` for the alert.
2. It's showing the error alert by presenting it on the view controller.

We could make this easier to test by extracting the message setting part into it's own function like this.

```swift
private func displayError(_ error: NetworkError) {
    let titleAndMessage = titleAndMessage(for: error)
    self.showErrorAlert(title: titleAndMessage.0, message: titleAndMessage.1)
}

func titleAndMessage(for error: NetworkError) -> (String, String) {
    let title: String
    let message: String
    switch error {
    case .serverError:
        title = "Server Error"
        message = "We could not process your request. Please try again."
    case .decodingError:
        title = "Network Error"
        message = "Ensure you are connected to the internet. Please try again."
    }
    return (title, message)
}
```

And then we could write a unit test for it like this.

**AccountSummaryViewControllerTests**

```swift
func testTitleAndMessageForServerError() throws {
    let titleAndMessage = vc.titleAndMessage(for: .serverError)
    XCTAssertEqual("Server Error", titleAndMessage.0)
    XCTAssertEqual("We could not process your request. Please try again.", titleAndMessage.1)
}
```
 
Run this test, and the test will pass 🎉.

### Challenge

See if you can write the corresponding test for `decodingError`.

- Copy the previous test we just wrote.
- Adjust the expectations in the assert.
- And cover the other test case.

### Solution

**AccountSummaryViewControllerTests**

```
func testTitleAndMessageForNetworkError() throws {
    let titleAndMessage = vc.titleAndMessage(for: .decodingError)
    XCTAssertEqual("Network Error", titleAndMessage.0)
    XCTAssertEqual("Ensure you are connected to the internet. Please try again.", titleAndMessage.1)
}
```

OK that wasn't too bad. One thing to note with unit tests is that unit tests are a form of *coupling*.

Coupling means that if you change you code, you need to change your test. The tighter the coupling, the more fragile the test.

What we've done here is fine, we just need to be aware that whever something changes the language in our warning, our test is going to break.

Which we may or may not want.

If you want to test that the right message is being displayed, but you want your unit test to be more loosely coupled, we could write an equivalent test like this.

```swift
func testTitleAndMessageForServerErrorLessCoupling() throws {
    let titleAndMessage = vc.titleAndMessage(for: .serverError)
    XCTAssertTrue(titleAndMessage.0.contains("Server"))
    XCTAssertTrue(titleAndMessage.1.contains("could not process"))
}
    
func testTitleAndMessageForNetworkErrorLessCoupling() throws {
    let titleAndMessage = vc.titleAndMessage(for: .decodingError)
    XCTAssertTrue(titleAndMessage.0.contains("Network"))
    XCTAssertTrue(titleAndMessage.1.contains("Ensure you are connected"))
}
```

Now the message is free to change, but our tests are less brittle. 

### Trading off encapsultion for testability

One other thing you may or may not of noticed here is we made the extracted function non-private.

```swift
func titleAndMessage(for error: NetworkError) -> (String, String) {...}
``

Normally when we are writing Object-Oriented (OO) code we try to follow the maxium of keep everything we can private.

In this case we choose to sacrifice some privateness for testability.

This is a trade-off we often make when testing UI related code. Trading off encapsulation for testing.

If we really wanted to keep this private we could. We could add an extension method purely for unit testing that would give us access to this internal method.

```swift
// Unit testing
extension AccountSummaryViewController {
    func titleAndMessageForTesting(for error: NetworkError) -> (String, String) {
            return titleAndMessage(for: error)
    }
}
```

Or extracted this logic into another component with a public method.

I am generally OK with opening things up for testing. I find it makes the code cleaner and easier to read. But if you or others you are working with are sticklers for OO, feel free to add extension methods like this one above and keep your internals private.

### Creating instance variables of the things you want to test

In cases like this, one simple trick for getting access for things you want to test is to make them instances variables in the view controller, and the access them in your test after.

For example, if we wanted to verify that an alert pops up with an error message is passed, we could by making the `UIAlertController` and variable like this.

```swift
// Error alert
lazy var errorAlert: UIAlertController = {
    let alert =  UIAlertController(title: "", message: "", preferredStyle: .alert)
    alert.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
    return alert
}()
```

And then setting it when an error occurs like this.

```swift
private func showErrorAlert(title: String, message: String) {
//        let alert = UIAlertController(title: title,
//                                      message: message,
//                                      preferredStyle: .alert)
//
//        alert.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
    
    errorAlert.title = title
    errorAlert.message = message
    
    present(errorAlert, animated: true, completion: nil)
}
```

We can first manually test that everything still works.

But now we can write a unit test for the alert like this.

```swift
func testAlertForServerError() throws {
    stubManager.error = NetworkError.serverError
    vc.forceFetchProfile()
    
    XCTAssertEqual("Server Error", vc.errorAlert.title)
    XCTAssertEqual("We could not process your request. Please try again.", vc.errorAlert.message)
}

```

Discussion:

- What is `loadViewIfNeeded()` and what it does in unit tests
- Unit testing is a real art (being doing for years, many languages and still learning).

Two maximum that help:

1. Break thing down into smaller pieces.
2. Dependency inject the things you want to change.



## OLD

### Leveraging protocols in our unit tests

Let's create a new file called `ProfileNetworkingTests`.

![](images/5.png)

And let's start by writing a happy path scenario for what we expect to happen when `AccountSummaryViewController` calls `fetchProfile`.

**ProfileNetworkingTests**

```swift
import Foundation

import XCTest

@testable import Bankey

class ProfileNetworkingTests: XCTestCase {
    var vc: AccountSummaryViewController!
    
    override func setUp() {
        super.setUp()
        vc = AccountSummaryViewController()
        vc.loadViewIfNeeded()
    }
    
    func testFetchProfile() throws {
        
    }
}
```

What we really want to test here, is that the `profile` gets set after we successfully do a fetch.

**AccountSummaryViewController**

```swfit
case .success(let profile):
    self.profile = profile
```

The problem is we have no way of triggering it from our test. Let's make `AccountSummaryViewController` a little more testable by extracting `fetchProfile` and `fetchAccounts` into their own public methods.

**AccountSummaryViewController**

```swift
fetchProfile(group: group, userId: userId)
fetchAccounts(group: group, userId: userId)
self.reloadView()

private func fetchProfile(group: DispatchGroup, userId: String) {
    group.enter()
    profileManageable.fetchProfile(forUserId: userId) { result in
        switch result {
        case .success(let profile):
            self.profile = profile
        case .failure(let error):
            self.displayError(error)
        }
        group.leave()
    }
}
    
private func fetchAccounts(group: DispatchGroup, userId: String) {
    group.enter()
    fetchAccounts(forUserId: userId) { result in
        switch result {
        case .success(let accounts):
            self.accounts = accounts
        case .failure(let error):
            self.displayError(error)
        }
        group.leave()
    }
}

private func reloadView() {
    self.tableView.refreshControl?.endRefreshing()
    
    guard let profile = self.profile else { return }
    
    self.isLoaded = true
    self.configureTableHeaderView(with: profile)
    self.configureTableCells(with: self.accounts)
    self.tableView.reloadData()
}
```

And then add a unit testing extension to access them publically.

```swift
// MARK: Unit testing
extension AccountSummaryViewController {
    func forceFetchProfile() {
        fetchProfile(group: DispatchGroup(), userId: "1")
    }
}
```

Discussion:

- Why the unit testing extension?
- Trade-offs of OO and testability
- Trading encapsulation for testability.

Now that our view controller is a little more testable, let's start with the happy path scenario of simply calling `fetchProfile` and verifying it sets the profile it returns to non-nil.

**ProfileNetworkingTests**

```swift
func testFetchProfile() throws {
    vc.profile = nil
    
    vc.forceFetchProfile()
    XCTAssertNotNil(vc.profile)
}
```

OK - if we run this now our test fails. Why? Because we are trying to do an asynchronous HTTP call. The way the view controller is configured, it is using the real `ProfileManageable` object that goes out and does the network call. We don't want that.

Unit tests that make network calls aren't really unit tests. They are more integration tests. Which are valuable. They just aren't the kind of test we want to write here.

What we want instead is something that is:

- Deterministic
- Not flakey
- Can be run reliably
- And is fast

That's kind of what a unit test is. Something fast, that doesn't rely on external dependencies. And can be run over-and-over again and never fail.

This is where our dependency-injection comes in. 

We can swap out the real network call with a fake one by defining a `StubProfileManager`, inserting it into the view controller in the unit test, and then control what happens from there.

Let's start by adding a `profileManager` to the test.

**ProfileNetworkingTests**

```swift
var vc: AccountSummaryViewController!
var stubManager: StubProfileManager! //
```

Then let create a stub implementing `ProfileManageable` to return hard coded values synchronously of whatever want - in this case a fake profile.

```swift
struct StubProfileManager: ProfileManageable {
    var profile = Profile(id: "1", firstName: "FirstName", lastName: "LastName")
    
    func fetchProfile(forUserId userId: String, completion: @escaping (Result<Profile, NetworkError>) -> Void) {
        completion(.success(profile))
    }
}
```

Explain what this is doing.

Then we can inject it into our view controller in the setup.

```swift
override func setUp() {
    super.setUp()
    vc = AccountSummaryViewController()
    
    stubManager = StubProfileManager() //
    vc.profileManageable = stubManager //
    
    vc.loadViewIfNeeded()
}
```

Now when we run out tests they pass, because we are using the stub instead of the real network call manager.

✅ Tests pass

Discussion:

- How we make this synchronous
- Stub vs Mock

## Testing for errors

OK. Not bad. Happy path works. What we really want though are error conditions. Want to test that an alert pops up when these two errors occur:

```swift
enum NetworkError: Error {
    case serverError
    case decodingError
}
```

Fortunately, we've done the heavily lifting, and can now leverage our stub to return an error instead of a profile.

First let's add an optional error to our stub. And if the error is present, return it instead of the profile.

**ProfileNetworkingTests**

```swift
struct StubProfileManager: ProfileManageable {
    var profile = Profile(id: "1", firstName: "FirstName", lastName: "LastName")
    var error: NetworkError? // 
    
    func fetchProfile(forUserId userId: String, completion: @escaping (Result<Profile, NetworkError>) -> Void) {
        if let error = error { //
            completion(.failure(error))
        }
        completion(.success(profile))
    }
}
```



### What we've learned

- 💥  How to manually test for network errors
- ⛑  How to fix them with pop-ups and alerts
- 🚀  How to unit test network code
- 🌟 How to build a more robust application

### Links that help

- [UIAlertController](https://developer.apple.com/documentation/uikit/uialertcontroller)
- [UIAlertControllerExample](https://github.com/jrasmusson/ios-starter-kit/blob/master/basics/UIAlertController/UIAlertController.md)
- [Mocks vs Stubs](https://martinfowler.com/articles/mocksArentStubs.html)