---
layout: post
title: "Dive into transaction timeout issue"
# subtitle: "This is the post subtitle."
comments: true
date:   2023-09-07 11:52:21 +0800
categories: 'en'
---

Write DB record(s) concurrently or parallelly is a common situation, while it can easily result in defects which is hard to detect in PR review and often cause disasters in production. My previous blog mentioned a deadlock issue which is caused by such scenario, briefly speaking the root cause is that we are trying to update/delete same batch records parallelly in different order and locked in mutual finally.

Recently we received another production issue which says `transaction rolled back by lock wait timeout: Lock-wait time out exceeded [OWNER=152260476930, TYPE=RECORD_LOCK, CURRENT_MODE=NON_KEY_EXCLUSIVE, REQUESTED_MODE=NON_KEY_EXCLUSIVE], rc=4628`. Lock wait timeout is better than deadlock, to be honest, it often indicates that there is a long running transaction and locked table(s) or row(s) so that other transactions cannot write it, btw, we rarely see that read transaction is blocked since modern database often provide MVCC mechanism to guarantee consistency read, unless you choose to use locking read.

Let's back to our topic, with a first glance at the error log and we can know lock owner transaction id , so first thing we did is to query history running statistics:

```sql
select * from _sys_statistics.host_blocked_transactions
where lock_owner_update_transaction_id = 152260476930
```

And we get over one hundred locked transaction records, after investigated the result, we knew what the situation is: one user was trying to update the records several times while our job is holding the record lock, all of those updates finally failed with timeout.

Now the question is why our job is keeping the lock for over 30 mins(the default timeout in HANA)? We went to our job monitor tools and noticed our daily job took 49 mins to finish in that day, sounds like it's clear? Definitely not, we did some optimization in the past to separate our DB operations in pages and each page takes one standalone transaction, that means even though the total job execution time is 49 mins, for each page it shouldn't take so much time, at least shouldn't over 30 mins, right? So we went to our code and check it again: (some unnecessary code omitted)

```javascript
  let ptx = undefined;
  try {
    const inactiveProgress = await getAllInactiveProgress(daysInactive);
    const total = inactiveProgress.length;
    do {
      ptx = cds.tx();
      page = pagination.getPage(inactiveProgress, offset, batchRows);
      if (page.length <= 0) {
        await ptx.commit();
        break;
      }
      const results = await notify(page);
      await update(results.success);
      await ptx.commit();
      offset += batchRows;
    } while (offset < total);
  } catch (error) {
    await ptx.rollback(error);
  }
```

Everything looks quite good, we are following SAP CDS explicit transaction management well, then we decided to reproduce it in our local, we added a sleep sentence after the do-while clause and at the meantime we sent a user request to update overlapping record, then we found that the user request got stuck, that means the defect is reproducible! (There is a small episode actually, when I set the column which needs to be updated to NULL and it's not locked, I assume it's something related to HANA column table implementation)

We add `DEBUG=ALL` and restarted our server locally to print transaction BEGIN/COMMIT logs, this time we found that there is only one BEGIN and COMMIT printed which means the `cds.tx()` in loop is not working, and we only have one single transaction per job, this explained our production issue. But wait, then what's wrong with our code? 

We rechecked CDS documentation and our code implementation and noticed a minor difference, in our code when invoking update we are using `cds.run(..)` while in CDS doc it's using `tx.run(..)`, we quickly realized that could be the cause of this defect, so we updated our code accordingly and restart the server to test, this time we found that everything is working as expected and each iteration successfully managed its own transaction.

Now the last question is what's the difference between `cds.run(..)` and `tx.run(..)`, unfortunately we didn't find any information is CAP documentation, so this time we dive into the implementation of cds nodejs package, cds is actually a facade and is defined in `@sap/cds/lib/index.js`, we could see that `cds.run` is a shortcut for `cds.db.run`:

```javascript
// cds as shortcut to cds.db -> for compatibility only
extend (cds.__proto__) .with ({
  get entities(){ return (cds.db||_missing).entities },
  transaction: (..._) => (cds.db||_missing).transaction(..._),
  // here is the cds.run defined
  run:         (..._) => (cds.db||_missing).run(..._),
  foreach:     (..._) => (cds.db||_missing).foreach(..._),
  stream:      (..._) => (cds.db||_missing).stream(..._),
  read:        (..._) => (cds.db||_missing).read(..._),
  create:      (..._) => (cds.db||_missing).create(..._),
  insert:      (..._) => (cds.db||_missing).insert(..._),
  update:      (..._) => (cds.db||_missing).update(..._),
  delete:      (..._) => (cds.db||_missing).delete(..._),
  disconnect:  (..._) => (cds.db||_missing).disconnect(..._),
})
```

`cds.db` is a type of `DatabaseService`, which is a child of `cds.Service`, when invoking `cds.run` it actually delegates the query parameter to `cds.Service.run`:

```javascript
/**
 * Querying API to send synchronous requests...
 */
run (query, data) {
  if (typeof query === 'function') {
    const ctx = cds.context, fn = query
    if (ctx?.tx && !ctx.tx._done) {
      return fn (this.tx(ctx)) // with nested tx
    } else {
      return this.tx (fn) // with root tx
    }
  }
  const req = new Request ({ query, data })
  return this.dispatch (req)
}
```

`cds.Service.dispatch` is using `srv-dispatch.js`, inside the code implementation we could see that by default it will use the context of current cds and execute the query within the context.tx:

```javascript
exports.dispatch = async function dispatch (req) { //NOSONAR

  // Ensure we are in a proper transaction
  if (!this.context) {
    const ctx = cds.context
    if (ctx?.tx && !ctx.tx._done) {
      return this.tx (ctx) .dispatch(req)      // with nested tx
    } else {
      return this.tx (tx => tx.dispatch(req))  // with root tx
    }
  }
  if (!req.tx) req.tx = this // `this` is a tx from now on...
  // ...omitted
  return this.handle(req)
}
```

From above we could find that the actual difference between `cds.run(..)` and `tx.run(..)`, briefly speaking that they are using different transaction(what a short answer).

### References

[1] https://help.sap.com/docs/SAP_HANA_PLATFORM/bed8c14f9f024763b0777aa72b5436f6/dd111c40e43c4161a59769c7e959a86b.html

[2] https://cap.cloud.sap/docs/node.js/cds-tx
