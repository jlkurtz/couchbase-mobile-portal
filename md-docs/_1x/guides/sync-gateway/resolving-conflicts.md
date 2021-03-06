---
id: resolving-conflicts
title: Resolving Conflicts
permalink: guides/sync-gateway/resolving-conflicts/index.html
---

A conflict usually occurs when two writers are offline and save a different revision of the same document. Couchbase Mobile provides features to resolve these conflicts, the resolution rules are written in the application to keep full control over which edit (also called a revision) should be picked. The [revision guide](http://developer.couchbase.com/documentation/mobile/current/develop/guides/couchbase-lite/native-api/revision/index.html) and [documents conflicts FAQ](http://developer.couchbase.com/documentation/mobile/current/develop/guides/couchbase-lite/native-api/document/index.html#document-conflict-faq) 
are good resources to learn how to resolve conflicts on the devices with Couchbase Lite. This guide describes how to handle the conflict resolution on the server-side using the Sync Gateway Admin REST API.

## Creating a conflict

During development, the **new_edits** flag can be used to allow conflicts to be created on demand.

```bash
// Persist three revisions of user foo with different statuses
// and updated_at dates
curl -X POST http://localhost:4985/sync_gateway/_bulk_docs \
  -H "Content-Type: application/json" \
  -d '{"new_edits": false, "docs": [{"_id": "foo", "type": "user", "updated_at": "2016-06-24T17:37:49.715Z", "status": "online", "_rev": "1-123"}, {"_id": "foo", "type": "user", "updated_at": "2016-06-26T17:37:49.715Z", "status": "offline", "_rev": "1-456"}, {"_id": "foo", "type": "user", "updated_at": "2016-06-25T17:37:49.715Z", "status": "offline", "_rev": "1-789"}]}'
            
// Persist three revisions of task bar with different names
curl -X POST http://localhost:4985/sync_gateway/_bulk_docs \
  -H "Content-Type: application/json" \
  -d '{"new_edits": false, "docs": [{"_id": "bar", "type": "task", "name": "aaa", "_rev": "1-123"}, {"_id": "bar", "type": "task", "name": "ccc", "_rev": "1-456"}, {"_id": "bar", "type": "task", "name": "bbb", "_rev": "1-789"}]}'
```

It can be set in the request body of the POST `/{db}/_bulk_docs` endpoint.

## Detecting a conflict

Conflicts are detected on the changes feed with the following query string options.

```bash
curl -X GET 'http://localhost:4985/sync_gateway/_changes?active_only=true&style=all_docs'
  
{
  "results": [
    {"seq":1,"id":"_user/","changes":[{"rev":""}]},
    {"seq":4,"id":"foo","changes":[{"rev":"1-789"},{"rev":"1-123"},{"rev":"1-456"}]},
    {"seq":7,"id":"bar","changes":[{"rev":"1-789"},{"rev":"1-123"},{"rev":"1-456"}]}
  ],
  "last_seq":"7"
}
```

With `active_only=true` and `style=all_docs` set, the changes feed excludes the deletions (also known as tombstones) and channel access removals which are not needed for resolving conflicts.

In this guide, we will write a program in node.js to connect to the changes feed and use the [request](https://github.com/request/request) library to perform operations on the Sync Gateway Admin REST API. The concepts covered below should also apply to other server-side languages, the implementation will differ but the sequence of operations is the same. In a new directory, install the library with npm.

```bash
npm install request
```

Create a new file called index.js with the following.

```javascript
var request = require('request');
var sync_gateway_url = 'http://localhost:4985/sync_gateway/';
var seq = process.argv[2];
  
getChanges(seq);
  
function getChanges(seq) {
  var querystring = 'style=all_docs&active_only=true&include_docs=true&feed=longpoll&since=' + seq;
  var options = {
    url: sync_gateway_url + '_changes?' + querystring
  };
  // 1. GET request to _changes?feed=longpoll&...
  request(options, function (error, response, body) {
    if (!error && response.statusCode == 200) {
      var json = JSON.parse(body);
      for (var i = 0; i < json.results.length; i++) {
        var row = json.results[i];
        var changes = row.changes;
        console.log("Document with ID " + row.id + " has " + changes.length + " revisions.");
        // 2. Detect a conflict.
        if (changes.length > 1) {
          console.log("Conflicts exist. Resolving...");
          resolveConflicts(row.doc, function() {
            getChanges(row.seq);
          });
          return;
        }
      }
      // 3. There were no conflicts in this response, get the next change(s).
      getChanges(json.last_seq);
    }
  });
}
```

Let's go through this step by step:

1. GET request to the _changes endpoint. With the following options:
  - **feed=longpoll&since=<seq>:** The response will contain all the changes since the specified seq. If seq is the 
 last sequence number (the most recent one) then the connection will remain open until a new document is processed by Sync Gateway and the change event is sent. The getChanges method is called recursively to always have the latest changes.
  - **include_docs:** The response will contain the document body (i.e. the current revision for that document).
  - **all\_docs&active\_only=true:** The response will exclude changes that are deletions and channel access removals.

2. **Detect and resolve conflicts.** If there are more than one revision then it's a conflict. Resolve the conflict 
and return (stop processing this response). Once the conflict is resolve, get the next change(s).
3. **There were no conflicts in this response, get the next change(s).**

The program won’t run yet because the resolveConflicts method isn’t defined, read the next section to learn how to resolve conflicts once they are detected.

## Resolving conflicts

To resolve conflicts, the open_revs=all option on the document endpoint returns all the revisions of a given document. The **Accept: application/json** header is used to have a single JSON object in the response (otherwise the response is in multipart format).

```bash
curl -X GET -H 'Accept: application/json' 'http://localhost:4984/sync_gateway/foo?open_revs=all'
```

From there, the App Server decides how to merge the data and/or elect the winning update operation. Add the following
 in `index.js` below the **getChanges** method.

```javascript
function chooseLatest(revisions) {
  var winning_rev = null;
  var latest_time = 0;
  for (var i = 0; i < revisions.length; i++) {
    var time = new Date(revisions[i].updated_at);
    if (time > latest_time) {
      latest_time = time;
      winning_rev = Object.assign({}, revisions[i]); //copy as a new object
    }
  }
  return {revisions: revisions, winning_rev: winning_rev};
}
  
function resolveConflicts(current_rev, callback) {
  var options = {
    url: sync_gateway_url + current_rev._id + '?open_revs=all',
    headers: {
      'Accept': 'application/json'
    }
  };
  // 1. Use open_revs=all to get the properties in each revision.
  request(options, function (error, response, body) {
    if (!error && response.statusCode == 200) {
      var json = JSON.parse(body);
      var revisions = json.map(function(row) {return row.ok;});
      var resolved;
      // 2. Resolve the conflict.
      switch (current_rev.type) {
        case "user":
          // Choose the revision with the latest updated_at value
          // as the winner.
          resolved = chooseLatest(revisions);
          break;
        case "list":
          // Write your own resolution logic for other doc types
          // following the function definition of chooseLatest.
        default:
          // Keep the current revision as the winner. Non-current
          // revisions must be removed even in this scenario.
          resolved = {revisions: revisions, winning_rev: current_rev};
      }
        
      // 3. Prepare the changes for the _bulk_docs request.
      var bulk_docs = revisions.map(function (revision) {
        if (revision._rev == current_rev._rev) {
          delete resolved.winning_rev._rev;
          revision = Object.assign({_rev: current_rev._rev}, resolved.winning_rev);
        } else {
          revision._deleted = true;
        }
        return revision
      });
        
      // 4. Write each change (deletion or update) to the database.
      var options = {url: sync_gateway_url + '_bulk_docs', body: JSON.stringify({docs: bulk_docs})};
      request.post(options, function (error, response, body) {
        if (!error && response.statusCode == 201) {
          console.log('Conflict resolved for doc ID ' + current_rev._id);
          callback();
        }
      });
    }
  })
}
```

So what is this code doing?

1. **Use open_revs=all to get the properties in each revision.**
2. **Resolve the conflict.** For user documents, the revision with the latest updated_at value wins. For other document types, the current revision (the one that got picked deterministically by the system) remains the winner. Note that non-current revisions must still be removed otherwise they may be promoted as the current revision at a later time. The resolution logic may be different for each document type.
3. **Prepare the changes for the \_bulk\_docs request.** All non-current revision are marked for deletion with the `_deleted: true` property. The current revision properties are replaced with the properties of the winning revision.
4. **Write each change (deletion or update) to the database.**

Start the program from sequence 0, the first sequence number in any Couchbase Mobile database.

```bash
node index.js 0
```

The conflicts that were added at the beginning of the guide are detected and resolved.

```
Document with ID _user/ has 1 revisions.
Document with ID foo has 3 revisions.
Conflicts exist. Resolving...
Conflict resolved for doc ID foo
Document with ID bar has 3 revisions.
Conflicts exist. Resolving...
Conflict resolved for doc ID bar
Document with ID foo has 1 revisions.
Document with ID bar has 1 revisions.
```

Add more conflicting revisions from the command-line with a different document ID (baz for example). The conflict is resolved and the program continues to listen for the next change(s).