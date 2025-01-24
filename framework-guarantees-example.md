# guarantees on the framework usage level

Let's discuss now guarantees that framework will provide for the user. For this I'll be using slightly simplier version of the storage config as in that example my focus would be on the UX for people who will write business-logic.

[TODO:?]Our goals are:
1. to provide guarantees that running the code is always safe from data definition's perspective. Meaning:
    - you'll have all required fields
    - they will have the correct type
2. you don't need to perform annoying migrations yourself

# Transition example

Let's imagine we have users, they have name and family name. Our code does something and in the end takes name+family-name and sends some confirmation email. And for some reason - we want to get rid of name and family name and want to start using name-and-family-name field. So we need to change the schema, perform the migration, expose the fields, update the code, remove old fields, etc.

## step 0
This is what we have for [storage configuration](./maroong-storage-config.md)
```python
storage(
	types(
		User(
			v1[active](
				id int
				name str
				family_name str
				index (
					id: hash,
					name: btree
				)
			),
		)
	)
	repositories (
		postgresql(
			id: "store-for-everything",
			connectionParams: {...},
			 holdTypes(
				 User.v1(
					 id -> users.id,
					 name -> users.name,
					 family_name -> users.family_name,
				 ),
			 )
		)
	),
	location_rules(
		User.v1("store-for-everything"),
	)
)
```

this is the code we have on [maroon-runner](./maroon-runner.md)
```python
oltp_maroon {
	maroon def (userID: str) {
		# do some logic here
		# will be executed durably, etc, bla bla


		# send email to the user
		var user = User(storage, index.hash==userID)

		send_email(full_name: user.name + ' ' + family_name)
	}
}
```

## destination state
we want to add the field that combines name + family-name. And use it in our code

```python
storage(
	types(
		User(
			v1(
				id int
				name str
				family_name str
				index (
					id: hash,
					name: btree,
				)
			), 
			v2[active](
				id int
				full_name str # name + family_name
				index (
					id: hash,
					full_name: btree,
				)
			),
			migration(
				# what we want here - to leave user a possibility to traverse data back and forth
				# in case if we need to roll-back the changes whatever the reason

				# constructor
				v1_v2(obj: v1) -> v2 { 
					# side effects are allowed: idempotent! http/db/etc. calls
					return v2{
						full_name: obj.name + ' ' + obj.family_name,
						v1...
					}
				}
				# restructor
				v2_v1(obj: v2) -> v1 { 
					# side effects are not allowed
					# we need this to populate previous versions 
					# when we create never versions
					return v1{
						name: full_name.split(' ').first,
						family_name: full_name.split(' ').second,
						v2...
					}
				}
			)
		)
	)
	repositories (
		postgresql(
			id: "store-for-everything",
			connectionParams: {...},
			 holdTypes(
				 User.v1(
					 id -> users.id,
					 name -> users.name,
					 family_name -> users.family_name,
				 ),
				 User.v2(
					 id -> users.id,
					 full_name -> users.full_name,
				 ),
			 )
		)
	),
	location_rules(
		User.v1("store-for-everything"),
        User.v2("store-for-everything"),
	)
)
```

Keep in mind that at any particular point in time there is only one available type version. In the code you can't combine two versions

```python
oltp_maroon {
	maroon def (userID: str) {
		# do some logic here
		# will be executed durably, etc, bla bla


		# send email to the user
		var user = User(storage, index.hash==userID)
		send_email(full_name: user.full_name)
	}
}
```

## execution. Step by step(simplified)

- we have type User_v1 active and used by the code on [maroon-runner](./maroon-runner.md)
- create new type with added field in configurator - [admin::role action]
- send new config to maroon-runner to verify - [admin::role action]
- rejected. Reasons
    - no constructor-restructor
    - no repositories updates for the new type
        [TODO:?] do we want to ask admin/dev to provide that information explicitly? Probably that's ok for the first version. Later - would be nice to make these things in an automatic way
    - no location_rules updates for the new type
- admin::role make udpates and sends it for the verification
- maroon-runner accepts the new config
- maroon-runner starts background migration job
    - creates necessary indexes/columns/tables in repositories
    - creates User_v2 type with the state [constructing]
    - starts to populate data for v2 objects
        - for each migrated version adds v2 to supported versions in [maroon-source-of-truth](./maroon-source-of-truth.md)
    - v2 type is in the [constructing] state and v1 - [active]
    - [!] if here dev::role tries to deploy the code that uses v2 - they get an error, because v2 is in [construct] state
    - [!] here admin::role can see the migration progress
        - exposed through some telemetry channel
        - [TODO:?] admin panel is here?
    - background migration process finished
- now v2 is in [available] state
	- all the indexes created
	- all the data is migrated
- now you can change the code and make v2 - active
- dev::role make changes:
    - in the config v2 - becomes active, v1 - becomes available
    - in the code - now it should use only v2 constructors, fields
- admin::role pushes the changes to maroon-runner
- maroon-runner checks the correctness and accept the changes
- we still have two types: v1 and v2
    - that means we still have info in DB for v1 and v2 and can switch any time we want
    - it also means that when we create v2 object - we save enough data to create valid v1 object (by using v2_v1 restructor)
    - so we can change v1 and v2 between active/availble state back and forth
- if we sure that we don't need v1 anymore we can perform "compaction" operation. That operation will:
    - delete v1 type
    - accept absence of constructor/restructor info for v1
    - accept absence of  repositories and location_rules info for v1
    - starts background process of deleting not needed columns
    - [!] it's a destructive operation. After applying it - we can't go between v1-v2 anymore. The information is lost