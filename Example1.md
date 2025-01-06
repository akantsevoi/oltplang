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

```
