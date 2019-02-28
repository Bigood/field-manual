**Note:** Please complete [Chapter 2 - Managing Data](./02_Managing_Data.md) first. 

# Chapter 3 - Structuring your data

> or, "How you learned to stop worrying and love _nested databases_."

- [Adding a practice counter to each piece](#)
- [Utilizing your practice counter](#)
- [Adding a higher-level user database](#)
- [Updating your user profile](#)

## Adding a practice counter to each piece

Your users may want to keep track of their practice, at minimum how many times they practiced a piece. You'll enable that functionality for them by creating a new OrbitDB `counter` store for each piece, and creating a few new functions inside the `NewPiecePlease` class to interact with the counters.

> **Note:**  The nesting approach detailed here is but one of many, and you are free to organize your data as you see fit. This is a powerful feature of OrbitDB and we are excited to see how people tackle this problem in the future!

Update the `addNewPiece` function to create a `counter` store every time a new piece is added to the database. You can utilize basic access control again to ensure that only a node with your IPFS node's ID can write to it.

```javascript
async addNewPiece(hash, instrument = "Piano") {
  const options = { accessController: { write: [this.orbitdb.identity.publicKey] }}
  const dbName = "counter." + hash.substr(20,20)
  const counterDb = await this.orbitdb.counter(dbName, options)

  const cid = await this.piecesDb.put({
    hash: hash,
    instrument: instrument,
    counter: counterDb.id
  })

  return cid
}
```

In your application code this would look something like this:

```javascript
const cid = await NPP.addNewPiece("QmdzDacgJ9EQF9Z8G3L1fzFwiEu255Nm5WiCey9ntrDPSL", "Piano")
const content = await NPP.node.dag.get(cid)
console.log(content.value.payload.value)
```

Which will then output something like:

```json
{
  "hash":"QmdzDacgJ9EQF9Z8G3L1fzFwiEu255Nm5WiCey9ntrDPSL",
  "counter":"/orbitdb/zdpuAoM3yZEwsynUgeWPfizmWz5DEFPiQSvg5gUPu9VoGhxjS/counter.fzFwiEu255Nm5WiCey9n",
  "instrument":"Piano"
}
```

### What just happened?

You changed your code to add a new database of type `counter` for each new entry added to the database.

* `  const options = { accessController: { write: [this.orbitdb.identity.publicKey] }}` should be recognizable from Chapter 1. This sets options for the db, namely the `accessController` to give write access only to your node's ID, or public key. `
* `this.orbitdb.counter` creates a new counter type with `options` that provide a write ACL for your IPFS node
* `const dbName = "counter." + hash.substr(20,20)` prepends `counter.` to the truncated database name. See the note below.
* `this.piecesDb.put` is then modified to store the _address_ of this new database for later retrieval similar to the way you  stored media addresses in a previous chapter.
* `"counter":"/orbitdb/zdpuAoM3yZEwsynUgeWPfizmWz5DEFPiQSvg5gUPu9VoGhxjS/counter.fzFwiEu255Nm5WiCey9n"` in the output now reflects this change by storing the _address_ of the new DB for later retrieval and updating.

> **Note:** There is a limit of 40 characters on the names of the databases, and multihashes are over this limit at 46. We still need unique names for each of the databases created to generate unique addresses, so we trim down the hash and prepend
it with `counter.` to get around this limitation.

## Utilizing the practice counter

Now, add a few functions to `NewPiecePlease` that utilize the counters when necessary

```javascript
async getPracticeCount(piece) {
  const counter = await this.orbitdb.counter(piece.counter)
  await counter.load()
  return counter.value
}

async incrementPracticeCounter(piece) {
  const counter = await this.orbitdb.counter(piece.counter)
  const cid = await counter.inc()
  return cid
}
```

These can be used in your application code like so:

```javascript
const piece = NPP.getPieceByHash("QmdzDacgJ9EQF9Z8G3L1fzFwiEu255Nm5WiCey9ntrDPSL")
const cid = await NPP.incrementPracticeCounter(piece)
const content = await NPP.node.dag.get(cid)
console.log(content.value.payload)
```

That will `console.log` out something like:

```json
{
  "op":"COUNTER",
  "key":null,
  "value": {
    "id":"042985dafe18ba45c7f1a57db.........02ae4b5e4aa3eb36bc5e67198c2d2",
    "counters": {
      "042985dafe18ba45c7f1a57db.........02ae4b5e4aa3eb36bc5e67198c2d2":3
    }
  }
}
```

### What just happened?

You created and used two new functions to both read the value of, and increment a `counter`, another type of OrbitDB store.

* `await this.orbitdb.counter(piece.counter)` is a new way of using `this.orbitdb.counter`, by passing in an existing database address. This will _open_ the existing database instead of creating it
* `counter.load()` is called once in `getPracticeCount`, loading the latest database entries into memory for display
* `await counter.inc()` increments the counter, like calling `counter++` would on an integer variable
* `"op":"COUNTER"` is a new operation that you havent seen yet - remember, you can create stores with any operations you want. More on this in Part 3.
* `"counters": { "042985dafe18ba45c7f1a57db.........02ae4b5e4aa3eb36bc5e67198c2d2": 3 }` is the value returned, the long value is an id based on your node's public key

## Adding a higher-level database for user data

Pieces of music to practice with are great to have, but moving forward you will want to allow users to further express themselves via a username and profile. This will also help prepare you for allowing users to connect to each other in the next chapter.

You will create a new database for users, from which your `piecesDb` will be referenced. You can create this database in the `ready` event handler of IPFS, alongside where you declared `piecesDb`.

```javascript
```

### What just happened?

You created a database to store anything and everything that might pertain to a user, and then linked the `piecesDb` to that, nested inside.

## Key Takeaways

* The distributed applications of the future will be complex and require data structures to mirror and manage that complexity.
* Luckily, OrbitDB is extremely flexible when it comes to generating complex and linked data structures
* These structures can contain any combination of OrbitDB stores - you are not limited to just one.
* You can nest a database within another, and you can create new databases to next your existing databases within.
* _Nesting_ databases is a powerful approach, but it is one of many. **Do not** feel limited. **Do** share novel approaches with the community.

And with this, you are now ready to connect to the outside world. Continue to [Chapter 4: Peer to Peer](04_P2P.md) to join your app to the global IPFS network, and to other users! 