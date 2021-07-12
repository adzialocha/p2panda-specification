# Schemas

In p2panda the users or the community define themselves how data should be structured. This gives plenty of possibilities of how the protocol can be used: For example to store chess movements for a game, festival events or posts in a social network. We call these blueprints of data structures *Schemas*, they describe how a certain kind of datum should be structured to be recognisable by everyone, like a "Festival event schema" or a "Chess movement schema".

Every author can propose a new schema to the network by publishing a special message which announces a new schema, for example `comment`:

```json
{
  "action": "create",
  "schema": "schema",
  "version": 1,
  "fields": {
    "name": "comment",
    "description": "Make a comment to a post",
    "version": 1,
    "fields": {
      "subject": {
        "type": "text"
      },
      "body": {
        "type": "text"
      },
      "createdAt": {
        "type": "timestamp"
      }
    }
  }
}
```

:bulb: *One could imagine a p2panda client / web application which offers a nice wizard-like UI to manage a schema step by step.*

The message format is following the same specification as all other [messages](#) with a few additional fields: The schema is described with a `name`, `description` and a `version` field, of which the latter is the version of the schema message format itself. Also it contains the definition of `fields` which each contain a `type`. The field names and types are used to verify incoming data from authors to check if their messages are well-formed and follow the schema specification.

Each schema helps to describe how data should be structured but also how it can be read and stored by the computer. This information can be used to easily build materialised view from Bamboo log entries (inspired by the [Kappa Database Architecture](http://milinda.pathirage.org/kappa-architecture.com/)).

## Register schemas

p2panda server administrators explicitly decide which schemas to install on them based on the needs of the community or themselves. Application developers can improve and extend their schemata by publishing updates, which are then again manually applied on servers. 

To use this newly introduced schema the administrator can register it via command line:

```
panda schema register --id a46863617...
🎍 Now indexing "comment" messages
```

This command would return an error if it couldn't find any schemas of type `comment` from this address.

The server checks if a schema of type `comment` is already registered. In this case it creates a temporary new table and runs the indexing mechanism, scanning all logs again to find messages of type `comment`.

Please note that the server will ignore all `comment` messages which do not follow the schema. This puts an extra responsibility on the creator of the schema and the administrator to not mess too much with breaking conventions. In case something important has to be changed clients have to be updated to support that change or a new schema name has to be introduced.

Schematas which refer to other schematas (relations), can be registered but are only indexed when all related schemes are registered as well. The server would register the schema and put them in some sort of waiting queue for indexing until all requirements are fulfilled.

We can imagine another user creating a scheme with the same name `comment`. The server admin addresses which schema is meant by referring to the specific schema entry id. For the client this distinction is not being made which puts a responsibility and community effort to maintain a common understanding on what names are used (similar to SSB and ActivityPub).

A server can be asked via the `panda_schemas` command to return all schemas it has currently registered.

## Multiple logs per user

Every user can maintain multiple bamboo logs. Every log has its own unique `log_id` (see https://github.com/AljoschaMeyer/bamboo#concepts-and-properties). Logs hold entries with different messages inside, but have one limitation: Each log can only hold messages of the same type. An log with multiple / mixed types is considered invalid and will be rejected by the server.

### Log types

The type of a log is defined by its first message. These logs are named **user logs** except of so called **system logs** which have pre-defined message types.

To separate the log types we define a numbering schema for the `log_id`:

* Even numbered ids `0, 2, 4, 6, ...` represent system logs
* Odd numbered ids `1, 3, 5, 7, ...` represent user logs

#### Type: System log

So far only system log `0` is defined in this specification but the other "free" system logs are kept for future protocol changes or new extensions (for example blob handling, name resolving, encryption etc.).

* System log `0` accepts messages of type `schema`

#### Type: User logs

No user log exists yet on the server when a client generates a new key pair. In the moment the clients sends its (first) message to the server, for example a create instance message of type `comment` ...

```json
{
  "action": "create",
  "schema": "comment",
  "version": 1,
  "fields": {
    "body": "Your concert was nice",
    "createdAt": "2020-05-31T22:51:04+0000"
  }
}
```

... the following happens:

1. Client asks server via RPC API (`panda_nextEntryArguments`) about the next `backlink` and `skiplink` for the next message of type `comment`. This is a required step to sign new bamboo entries on the clients side. The server looks into the database and checks if already any user log for `comment` exists. Since this is the very first message of this user, it can't find anything.

    :bulb: *The server would reject this request if messages of type `comment` are not accepted. For this example we assume they are accepted.*

2. The server picks the next unused user log id, which in this case is `1` (it is the first possible user log id) and sends it back to the client including `null` for the `backlink` and `skiplink` field as no entries exist yet.

    ```
    {
      "log_id": 1,
      "backlink_hash": null,
      "skiplink_hash": null
    }
    ```

3. With this data, the client can encode the new bamboo entry now and send it finally to the server (via `panda_publishEntry`).

    ```
    {
      "entryBytes": [...],
      "payloadBytes": [...]
    }
    ```

    This encoded entry contains all required information for the server, including the `log_id`.

4. After validating the bamboo entry integrity the server checks (again) if it accepts messages of type `comment`. It also checks if the message is well-formed and all fields are following the defined schema. (Schemas are stored in system log `0` but live in memory of the server to improve the speed of validation and be able to pre-compile regex checks.)

5. Since we know the `log_id` from the entry the server checks if there are already any existing entries and which type they have. If it is not `comment` the request will be rejected. In this example the log is empty so the entry will just be inserted into the database.

Now the user maintains a user log of type `comment` at log id `1`. All future `comment` messages will be placed in this log!

# TODO

- Correct RPC commands examples to follow latest API
- Mention Regular Expressions
