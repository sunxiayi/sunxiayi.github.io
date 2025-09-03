---
title: Learn by doing - Using AI to build an iOS app
date: 2024-09-02
categories: CS
tags:
- AI
- Claude Code
- Cursor
- Codex
- ChatGPT
---

I recently started building an app about healthy sleep habits. Of course, one MUST use vibe coding to prove they live in this era.. So I did. 

Since my last experience of iOS development is around 7 years ago, you can say my experience of iOS development is nil at this point. But luckily ramping up on Swift is not that hard, especially when you are learning it by building your own project. What's even better, nowadays AI is a great teacher showcasing basic codebases to you. So after I'm done with some basic functionalities, I started reviewing the code it writes. It is quite an interesting and rewarding journey when I'm reading code of my own project, and now AI comes much handy when you want to ask any questions.

Toolkit: 

- Claude Code: bootstrap the project, code skeletons, most of code written by it
- Cursor, Codex: small fixes, asking code questions
- ChatGPT: asking therotical or conceptual heavy questions

Here are some basics I learned quite smoothly:

### Basic Attributes

#### @State

```swift
@State private var showingError = false
@State private var routineName = ""
```

Purpose: Manages local state within a single view

When to use: For UI state that only this view needs

Lifecycle: Created and destroyed with the view

```swift
struct MyView: View {
    @State private var isToggled = false
    
    var body: some View {
        Toggle("Switch", isOn: $isToggled)
    }
}
```

#### @Binding

```swift
@Binding var isManualMorningMode: Bool?
```

Purpose: Creates a two-way connection between parent and child views

When to use: When you need to modify data owned by a parent view

Example:

```swift
// Parent view
struct ParentView: View {
    @State private var text = "Hello"
    
    var body: some View {
        ChildView(text: $text) // Pass binding
    }
}

// Child view
struct ChildView: View {
    @Binding var text: String
    
    var body: some View {
        TextField("Enter text", text: $text)
    }
}
```

#### @ObservableObject

```swift
class ReminderViewModel: ObservableObject {
    @Published var routines: [Routine] = []
}
```

Purpose: Makes a class observable to SwiftUI views

When to use: For complex data models that multiple views need

Example:

```swift
class UserManager: ObservableObject {
    @Published var isLoggedIn = false
    @Published var currentUser: User?
    
    func login() {
        isLoggedIn = true
        // Views automatically update
    }
}
```

#### @Published

```swift
@Published var routines: [Routine] = []
@Published var targetSleepTime: Date = Date()
```

Purpose: Makes properties observable in ObservableObject classes

When to use: Inside ObservableObject classes to notify views of changes

Example:

```swift
class ReminderViewModel: ObservableObject {
    @Published var routines: [Routine] = []
    
    func addRoutine(_ routine: Routine) {
        routines.append(routine) // Automatically notifies views
    }
}
```

#### @EnvironmentObject: 

```swift
@EnvironmentObject var reminderViewModel: ReminderViewModel
```

Purpose: Accesses shared data injected into the environment

When to use: When multiple views need access to the same data

Example:

```swift
// App level
ContentView()
    .environmentObject(ReminderViewModel())

// Any child view
struct SomeView: View {
    @EnvironmentObject var viewModel: ReminderViewModel
    
    var body: some View {
        Text("Routines: \(viewModel.routines.count)")
    }
}
```

#### @AppStorage

```swift
@AppStorage("notificationsEnabled") var notificationsEnabled: Bool = true
```

Purpose: Automatically saves/loads data to/from UserDefaults

When to use: For user preferences and settings

Example:

```swift
struct SettingsView: View {
    @AppStorage("darkMode") var darkMode = false
    @AppStorage("username") var username = ""
    
    var body: some View {
        Toggle("Dark Mode", isOn: $darkMode)
        TextField("Username", text: $username)
    }
}
```

| Property Wrapper   | Use When                               |
| :----------------- | :------------------------------------- |
| @State             | Local UI state (toggles, text fields)  |
| @Binding           | Child views need to modify parent data |
| @Published         | Properties in ObservableObject classes |
| @EnvironmentObject | Multiple views need same data          |
| @AppStorage        | User preferences that persist          |
| @ObservableObject  | Complex data models                    |



### Declarative SwiftUI

SwiftUI framework is very powerful. Developers only need to use the corerct property wrappers and the internals are handled by the framework(hence a declarative UI framework). For example, when a @State or @Published variable changes, the view gets re-rendered automatically. I am very curious on how this is handled and ask ChatGPT on that. The update cycle is like this.

1. Mutation happens. `@State` write, or an `@Published` property sets and sends `objectWillChange`.
2. Invalidation & coalescing. SwiftUI marks dependent nodes dirty and coalesces work into a transaction for the current run loop turn.
3. Re-evaluation. SwiftUI reevaluates `body` on invalidated identities, producing a new value-tree.
4. Diff. It diffs old vs. new view values to compute a minimal set of mutations to the persistent render objects.
5. Apply & render. Changes are applied, only invalidated regions get updated.

In short, property wrappers register dependencies. Mutations invalidate just the views that depend on them. SwiftUI re-evaluates `body`, diffs results, and applies minimal mutations to the real render tree—usually via Combine (`objectWillChange`) for reference-typed models and via internal state storage for value-typed `@State`. The big three pillars to keep in mind are [identity, lifetime, and dependencies](https://developer.apple.com/videos/play/wwdc2021/10022/). 



### Background Task

In my app, I have a need to send a long background audio at a scheduled time even when the app is in the background. By the documentation:

- When the app is in the background, only 30s audio can be played.
- When the app is in the foreground, longer audio can be played. However, if the app is in the background, the scheduled event won't trigger the audio.

That would facilitate some innovation in implementation. At first, Claude Code implements a version when a background task is fired every 30 seconds to check for scheduled reminder. Then, I ask it to find potential performance issues in the code, and this one is recognized as "battery draining". 

I then switch to Codex to ask for the same performance improvement, this one is also recognized. It then recommends another approach to just wake up the task around deadlines of the reminder. Since this wakeup is not exact(BGAppRefresh is opportunistic, UNCalendarNotificationTrigger is exact), I further work with Codex to handle the case where BGAppRefresh is fired before and after UNCalendarNotificationTrigger to avoid duplicate audio playing. In this process, I play the role of suggesting the deduplication problem, solution and edge cases, Codex implements it.

From this experiment, I find Codex by default its implementation is accurate, but does not have a lot of "ownership" - you need to give it very specific instructions to do things, it won't go beyond my expectations, also often miss multi-file context. 