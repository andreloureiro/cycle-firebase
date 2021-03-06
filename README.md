# Cycle Firebase driver

Thin layer around the firebase javascript API that allows you to query and
declaratively update your favorite real-time database.

# Installation

```
$ npm install cycle-firebase
```

# Usage Example

```
import Cycle from '@cycle/core'
import {makeFirebaseDriver} from 'cycle-firebase'
import Firebase from 'firebase'

let main = ({firebase}) => {
  // ... Code that uses firebase driver
}

Cycle.run(main, {
  firebase: makeFirebaseDriver(new Firebase(YOUR_FIREBASE_URL)),
  // ... Other drivers
})

```

# API

I am not that good at describing API's, so PRs or issues are very welcome here :-)

## Source
The source consists of a couple of methods that will yields observables,
like most cycle driver sources ;)

#### `.get(location: String): Observable<Any>`
Returns an observable over the value on a specific location.
Will yield null, just like firebase, if there is nothing set.
This location is relative to the location of the firebase source
object (so `.child` affets this method).

There are two special locations you can listen for as well. Those are not
coming from firebase, but provide info about the status of firebase in your app:

**`$lastError: Observable<Error>`**
Right now, is an observable over the last error emitted by the authentication
part of your app. I am still thinking about putting that in the same observable
as the user info itself (see next special location), but I figured it would be
fine putting it here as well. It contains error objects from firebase itself.

```
firebase.get(`$lastError/message`).map(message => `An error occurred: ${message}`)
```

**`$user: Observable<Null|User>`**
An observable over the current logged in user, or null. Use this instead of the
now deprecated `firebase.uid$`. `User` here, is the 'raw' object emitted by
firebases' `onAuth` method. It will emit once on subscription.

```
let isLoggedIn$ = firebase.get(`$user`).map(user => user !== null)
```


#### `.child(location: String): FirebaseDriverSource`
Returns object similar to the original source, only scoped
to the location specified, just like `firebaseRef.child`.
Useful if you want a component to be bound to a smaller scope of data.

#### `.ref(): FirebaseRef`
Returns the firebase ref scoped to the same path as the firebase source.
Useful in combination with `.value` and `.observe` as you'll see...

#### `.pushId$: Observable<String>`
Observable that will yield one random generated, firebase compatible, Id.
This Id is generated exactly the way firebase would generate the Id for you,
so you can use it safely as a key:

```
let someAction$ = ...

let listWithRandomKeys$ = someAction$.flatMapLatest(x => {
  return firebase.pushId$.map(id => {
    return {
      [id]: x,
    }
  })
})

return {
  firebase: listWithRandomKeys$.map(list => {
    return { list }
  })
}
```

#### `.value(FirebaseRef|FirebaseQuery): Observable`
Will transform a firebase query or ref into an observable over it's
value. Useful for creating queries more complex than just a location.

```
let userQuery = firebase.child('users').ref().orderByChild('age').equalTo('19')
let user19yo$ = firebase.value(userQuery).map(...)
```

#### `.observe(FirebaseRef|FirebaseQuery, eventName): Observable`
If you, for some weird reason, need to use any of the `child_xxx` events,
`.observe` is for you. It wraps the event `eventName` in a very simple way
so it becomes an observable. Most of the time, though, you want to use `.value`!

#### `.$set(data: Any): Object`
Just wraps an object in `{ $set: object }` so the sink will
recognize it as an overwrite of data, not an update.

## Sink
The sink expects the be fed object, describing the way you want the firebase DB to look.
It will assume any unmentioned key will be untouched, except for object from `.$set`.
It will look at the changes every time, and update the new keys.
For removal, you'll have to explicitly set a key to null. Just not including the
key anymore, won't make the driver remove it.

#### Special location `$user`
The `$user` location does not get mirrored in firebase, but instead updates
the current user logged in. This can be done by setting (with $set!!!) the location
to an object with a `type` property, and potentially more options.
The possible types are:

- **{ type: 'password', email, password }** uses `authWithPassword`
- **{ type: 'password', create: true, email, password }** creates a user using `createUser` followed by `authWithPassword`
- **{ type: 'token', token }** uses `authWithCustomToken`
- **{ type: 'anonymously' }** uses `authAnonymously`
- **{ type: 'oauth_popup', provider }** uses `authWithOAuthPopup`
- **{ type: 'oauth_token', provider, token }** uses `authWithOAuthToken`

#### Examples
If the object changed from
```
{
  value: 1,
}
```
to
```
{
  value: 2
}
```

it will initially set `value` to `1`, (transition from `{}` to `{value: 1}` but
when the second object enters, it will, as one might expect, update the key
`value` to `2` on firebase.


If the object changed from
```
{
  users: {
    user1: {
      name: 'Michiel Dral',
    }
  },
}
```
to
```
{
  users: {
    user2: {
      name: 'Jake',
    }
  },
}
```

It will initially update the key `users/user1/name` to `Michiel Dral`, and then it
will update the key `users/user2/name` to `Jake`. Notice that it will
1. Not update the `user1` location after we drop it (doesn't remove it)
2. Not alter any other key than `users/user1/name` and `users/user2/name`

If you want to overwrite (and not update, like happening normally) a value on
a key with an object, you can use `firebase.$set` like this:

```
{
  users: {
    user2: firebase.$set({
      name: 'Jake',
    })
  },
}
```

this will wipe all the content of `users/user2` and set it to just `{ name: 'Jake' }`


# Contributing

I tried to comment the code as much as I could, and made the tests clear as well,
there are indeed no tests for the source yet, I couldn't be bothered just yet.
The styling is strict, and I want everything to obey it, so if something doesn't
fit a styling rule, we'll remove the styling rule :P
If you have a issue, please open one, or if you just want to discuss some stuff!
I have never received pull requests for a project yet, so I am thrilled to do so(:

- Michiel Dral
