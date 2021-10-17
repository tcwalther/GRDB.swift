GRDB Concurrency
================

**This guide describes how GRDB helps your app deal with database concurrency.**

Concurrency is hard, and better avoided when you can. But sometimes your application will need it.

Concurrent database accesses happen, for example, when the database file is shared accross several processes. Maybe you develop an iOS app that communicates with an extension through a shared database? This, honestly, is the most difficult setup, and there exists a dedicated [Sharing a Database] guide about it.

Other apps have strict performance requirements, and do not want to feel unresponsive because a slow database accesses stalls the user interface. Such apps need database concurrency in order to move slow database jobs off the main thread.

So please follow the [GRDB Concurrency Rules], right from the start. They do not make your app more complex than necessary. They can be applied by all developers, regardless of their skills and experience. They help your app gracefully evolve over time, and become more complex when needed.


## GRDB Concurrency Rules

There are only two rules, and they are at the root of safe GRDB concurrency.

<a id="rule-1"></a>:point_up: **Rule 1: Connect to any database file only once**

Open one and only one [DatabaseQueue] or [DatabasePool] per database file, for the whole duration of your use of the database. Not for the duration of _each_ database access, but really for the duration of _all_ database accesses.

:bulb: *Practical advice* - An app that uses a single database will connect only once. A document-based app will connect each time a document is opened, and disconnect when the document is closed. See the [demo apps] in order to see how to setup a UIKit or SwiftUI application for a single database.

:bulb: *If you do not follow rule 1:*

- You will not be able to use the [Database Observation] features.
- Everything will run correctly until your app starts using Swift async/await, or multi-threaded database accesses. You will see SQLite errors [SQLITE_BUSY].

<a id="rule-2"></a>:point_up: **Rule 2: Group related database operations within a single database access**

The game here is to group related database operations (fetches, inserts, updates) between a single pair of brackets: `read { ... }`,  or `write { ... }`.

For example, when you want to transfer money from one account to another, you instruct GRDB and SQLite to make sure both accounts are modified together. Storing a credit and a debit on disk are related operations, and you perform a single `write`:

```swift
// Group both transfers in a single database write
try dbQueue.write { db in
    try Debit(account: source, amount: amount).insert(db)
    try Credit(account: destination, amount: amount).insert(db)
}
```

Some database reads need to be grouped as well. When you display several database values on a screen, you want them to fit well together. To this end, you group related fetches in a single `read`, in which you are sure all requests see the same state of the database.

Generally speaking, within each database access, within each `read` or `write`, your app can create, maintain, and rely on your *database invariants*. A database invariant is a rule that your app must observe at all times, such as "all credits must have a matching debit", or "all books must have an author", etc. The technique that makes sure those invariants hold is to mind your [transaction boundaries](https://en.wikipedia.org/wiki/Concurrency_control#Database_transaction_and_the_ACID_rules), and this is precisely what `read` and `write` do.

:bulb: *Practical advice* - It helps to distinguish two kinds of methods: those that accept a `Database` arguments, and those that do not. The first ones may temporarily break invariants, but they can be composed and grouped together. The last ones can not be grouped, but they perform the high-level operations that preserve invariants. See sample code below:

<details><summary>Sample code</summary>

```swift
// MARK: - Methods that can be grouped

func register(_ db: Database, transfer: Transfer) throws {
    try Debit(account: transfer.source, amount: transfer.amount).insert(db)
    try Credit(account: transfer.destination, amount: transfer.amount).insert(db)
}

// MARK: - High-level operations that preserve invariants

// Preserved invariant: debit/credit pairs are always
// committed together on disk.
func register(transfer: Transfer) throws {
    try dbQueue.write { db in
        try register(db, transfer: transfer)
    }
}

// Preserved invariant: debit/credit pairs are always
// committed together on disk.
func revert(transfer: Transfer) throws {
    try register(transfer: transfer.reversed())
}

// Preserved invariant: transfer batches are always
// imported as a whole.
func import(transfers: [Transfer]) throws {
    try dbQueue.write { db in
        for transfer in transfers {
            try register(db, transfer: transfer)
        }
    }
}
```

Note how the built-in GRDB methods such as `fetchOne(db)`, `insert(db)`, etc. are methods of the first kind, because they accept a `Database` arguments, and they can be composed and grouped together. They are the building blocks of your own methods with a `Database` argument.

Now compare the above sample code with the fragile version below:

```swift
// OK
func register(transfer: [Transfer]) throws {
    try dbQueue.write { db in
        try Debit(account: transfer.source, amount: transfer.amount).insert(db)
        try Credit(account: transfer.destination, amount: transfer.amount).insert(db)
    }
}

// OK
func revert(transfer: Transfer) throws {
    try register(transfer: transfer.reversed())
}

// BROKEN INVARIANT, because related operations are not grouped.
// This commits a partial batch on disk if one transfer fails,
// or if the application crashes in the middle. This will
// creates duplicate transfers when you retry later.
func import(transfers: [Transfer]) throws {
    for transfer in transfers {
        try register(transfer: transfer)
    }
}
```

One other version, unfortunately fragile at every level:

```swift
// BROKEN INVARIANT: this commits on disk a debit without its
// matching credit.
func register(debit: Debit) throws {
    try dbQueue.write { db in
        try debit.insert(db)
    }
}

// BROKEN INVARIANT: this commits on disk a credit without its
// matching debit.
func register(credit: Credit) throws {
    try dbQueue.write { db in
        try credit.insert(db)
    }
}

// BROKEN INVARIANT, because related operations are not grouped.
// If the credit fail, or if the application crashes in the middle,
// the database contains a debit without its matching credit.
func register(transfer: [Transfer]) throws {
    try register(debit: transfer.debit)
    try register(credit: transfer.credit)
}
```

:point_right: Think about your database invariants, and make sure you take care of them. Nobody else can do it for you.

</details>

:bulb: *If you do not follow rule 1:*

- If your app crashes, you will find inconsistent data in your database on the next app launch. Good luck fixing this mess.
- When your app starts using [Database Observation], or Swift async/await, multi-threaded database accesses, or shares the database with another process, you will see broken database invariants. All it takes are database accesses that are interleaved because they were not grouped according to rule 2, such as reading an account balance right between the insertion of a debit and a credit.


[GRDB Concurrency Rules]: #grdb-concurrency-rules
[DatabaseQueue]: ../README.md#database-queues
[DatabasePool]: ../README.md#database-pools
[demo apps]: DemoApps
[Database Observation]: ../README.md#database-changes-observation
[SQLITE_BUSY]: https://www.sqlite.org/rescode.html#busy
[Sharing a Database]: SharingADatabase.md
[ValueObservation]: ../README.md#valueobservation
