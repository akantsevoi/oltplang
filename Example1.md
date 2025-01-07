Example of usage where we are combining marroon-daemon operations and oltp operations.
Since we're changing the data that is being migrated our engine keeps track of object's versions, keeps track of what has been migrated and what's not and where exactly the update should be written and committed.


```python

# migrates users and accounts from one db to another while maintaining OLTP queries

maroon-daemoon {
	for u in storeMDB.users {
		storePSQL.users.create(u, phantom: true)
		while storeMDB.accounts.countForUser(u.uid) > 0 {
			let accounts = storeMDB.accounts.get(u.uid, 10);
			#
			# some possible transformations here...
			#
			storePSQL.accounts.write(accounts);
		}
		storePSQL.users.update(u.uid, phantom: false);
	}
}

# oltp operations have higher priority than maroon-daemoon operations
oltp {
	# combinedStore - universal address space on top of 2 storages
	# keeps track of the object versions, knows where we have the updated version and queries the correct DB
	
	  combinedStore = {
		  storePSQL
		  storeMDB
	  }

	# "O(1) operation" - meaning we have limited amount of operations that will be finished "quickly"
	# transactional guarantees
	# 
	def payment(srcAcc: string, dstAcc: string, amount: int) {
		let tx = combinedStore.txStart();

		if combinedStore.accounts[srcAcc].amount <= amount {
			return InsufficientFunds{...}
		}
		
		combinedStore.accounts[srcAcc].amount -= amount;
		combinedStore.accounts[dstAcc].amount += amount;
		combinedStore.transfers.create(Transfer{
			From: srcAcc,
			To: dstAcc,
			Amount: amount,
			Time: now(),
		});

		tx.commit();
	}
}


#
# my fuzzy view on how it might look like
#

type Meta {
	version int64
	hash string

	# the information about which storage hold the up to date version of this object. The source of truth for that specific object
	storage string
}

type Object {
	Body # actual content
	Meta
}

impl combinedStore {
	storages: [](storage, options)
	objects: Map<id, meta>

	fn get(table: string, id: string) -> Object {
		# table == "Accounts" for example
		
		let m = objects[id];
		return storages.get(m.storage, table, "id == {id}")
	}

	fn writeNewVersion(obj: Object) {
		obj.meta.version++
		obj.meta.hash = hash(obj.body) 
		storages.get(obj.storage).Update(table,  obj) # unconditional write? since we control the sequence
	}
}

```


So, what happens in essense:
- we have long term process and short term process
- we can support consistent state only inside "short O(1)" operations
- but we need to support long-term scenarious as well
- in order to support these long-term scenarious we should be able to automatically slice our long-term scenarious into a sequence of correct/pausable "short O(1)" operations (are we talking about CRDT? again)
- then we just execute these operations with different priorities
	- ex: maroon operations have lower priority, and when we finish yet another cycle iteration we check if we have something with the higher priority -> for example "oltp request" and if we have - we execute higher priority thing, if not - continue maroon execution

Challenge:
- how to make sure that our slicing of operations will be ended up in the correct sequence of CRDT operations?
