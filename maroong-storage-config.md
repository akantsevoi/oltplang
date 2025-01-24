# Configuration language

- declarative approach
- strong type system
- indexes support
    - [TODO:?] do we have indexes on [source of turth](./maroon-source-of-truth.md) level? Or indexes will be spreaded to [repositories](./repository.md) as well? Or these are two different indexes?
- we need to clearly specify external repositories as our direction right now is to keep the customers data in their DBs
	- different or the same types of objects can live in the same or different repositories. Examples:
		- all the data in one storage
		- part of users live in mongo and part in postgresql and some special users can be created only in that AzureDB in that region due to regulations or whatever
	- admin can also add/remove repositories at "any" time
		- "any" - of course not any time, but almost. Limitations TBD
	- admin can change the parameters of repositories and maroon-engine will migrate the data between repositories accordingly

```python
storage(
    types(
		User(
			v1[active](
				id int
				name string
				country string
				index (
					name: btree
				)
			), 
			v2(
				id int
				name string
				country string
				active bool
				age int
				index (
					name: btree # btree because we'll need to query ranges
					age: hash # hash because we'll need to query exact values
				)
			)
			migration(
				# since new field is non-optional we need to add some code that can perform the transition between v1 and v2
				# underthehood engine will do the heavylifting:
				#   - introduce a new `active` optional field
				#   - starts updating value of the field
				#   - when finishes - it will move the column from optional to non-optional state
				v1_v2(obj: v1) -> v2 { 
					# not very declarative. Other proposals how to solve that situation?
					is_active := http.call.is_active(obj.id)
					if age := http.call.age_of_user(obj.id); age != nil {
						age = default_age
					}
					return v2{ # choose which fields to setup and which just copy from the old
						active: is_active,
						age: age,
						v1...
					}
				},
                # restructor
                # gets the previous version of the object
                # doesn't allow side effects
                # needed if we want to keep the possibility to roll back on severl versions
                v2_v1(obj: v2) -> v1{
                    return v1{
						v2...
					}
                }

			)
		)
	)
	repositories (
		mongodb(
			id: "eu-users-repository",
			connectionParams: {bla bla},
			 holdTypes(
				 User.v1(
					 id -> users.id, // mapping between in-memory object and table/field in a table datastore
					 name -> users.name,
					 country -> users.country,
				 ),
				 User.v2(
					 id -> users.id,
					 name -> users.name,
					 country -> users.country,
					 active -> users.active
				 ),
			 )
		),
		postgresql(
			id: "eu-users-repository-new", # imagine that we're migrating users from mongodb to postgres(unified storing approach), but it still should be in some EU-based DC
			connectionParams: {...},
			 holdTypes(
				 User.v1(
					 id -> users.id,
					 name -> users.name,
					 country -> users_meta.country (foreing_key: users.id), # compound object that lives in different tables
				 ),
				 User.v2(
					 id -> users.id,
					 name -> users.name,
					 country -> users_meta.country (foreing_key: users.id),
					 active -> users.active,
				 ),
			 )
		),
		postgresql(
			id: "us-users-repository",
			connectionParams: {...},
			 holdTypes(
				 User.v1(
					 id -> users.id,
					 name -> users.name,
					 country -> users_meta.country (foreing_key: users.id),
				 ),
				 User.v2(
					 id -> users.id,
					 name -> users.name,
					 country -> users_meta.country (foreing_key: users.id),
					 active -> users.active,
				 ),
			 )
		)
	),
	location_rules(
		priority_migration(
			# in that case data will be slowly copied from one storage to another
			# TODO: we need to have a requirement here that transformation should cover all the fields and it should be checked
			"eu-users-repository" ==> "eu-users-repository-new"
		)
		User.v1(country == "USA" => "us-users-repository"),
		User.v2(country == "USA" => "us-users-repository"),
		User.v1(country == "UK" => ["eu-users-repository", "eu-users-repository-new"]),
		User.v2(country == "UK" => ["eu-users-repository", "eu-users-repository-new"]),
	)
)
```