# `oltplang`

Obligatory disclaimers:

* The syntax itself can and likely will change.
* One underlying storage layer can be a SQL RDMBS. But it's by far not the only possible backbone.
* The language is designed to be transpiled. Think of these as `.proto` files, but for the logic, not for the schema.
* The snippets from higher-order language assume this higher-order language is C++.

# Example

## Define a `ProtoUser`

A `ProtoUser` is the user type with string `name` as the key (sic!), and int `age` as the only data field.
```
# Define the type. Self-explanatory.
# Note that the syntax does use curly braces.
# Semicolons and extra parentheses are avoided by design though.
# The language is case-sensitive.
# Keywords and types are UPPERCASE, as they should stand out.

TYPE ProtoUser {
  name STRING
  age INT
}
```

## Define the storage schema (`DB`) for the `ProtoUser`-s table

```
# Ensure that the database contains the table (dictionary) of users.
# The `DB` block can be used multiple times.
# Of course, in the real code, there will be a special "OLTP source" file for the DB schema.

DB {
  # The name of the table is `users`, and its type is `TABLE<User>`.
  # The `TABLE<T>` is effectively a dictionary, a.k.a. "just a hashmap".
  proto_users TABLE<ProtoUser> {
    # The only required argument for `TABLE` is `PRIMARY_KEY`.
    # Hard to have a dictionary without the primary key.
    PRIMARY_KEY name
    # NOTE(dkorolev): I'm contemplating whether it's better to support
    # some `PRIMARY_KEY` annotation right in the `TYPE` of `ProtoUser`.
  }
}
```

## Define PUT for `ProtoUser`-s

Note that PUT is not directly an HTTP verb, just a convenient term.

```

# The `COMMAND` syntax defined the mutation for the OLTP storage.
# It gets its own sequence ID, is executed serially, and is reproducible/replayable.
# For strongly consisteny reads, use `QUERY`.
# For eventually consistent reads -- TBD, but they can and should go through replicas.
COMMAND PutProtoUser(u ProtoUser) {
  # That's it. `PUT` is a keyword for the `DB` `TABLE`.
  # And the type matches, i.e. all the fields are there.
  # So this code is correct by itself.
  PUT users u
  # Had the type of `u` been different (for example, if it is constructed on the fly),
  # mis-typed, and/or missing non-optional fields would result in static type error.
}
```

## Use the above from outside

The above is a DSL, which is transpiled to be used natively from higher-level languages. For instance, from Java/C++:

```
OLTP.PutProtoUser(OLTP.ProtoUser().name("John Doe").age(42));
```

## Give users IDs

And make it impossible to PUT a user with an existing ID.

```
# A dedicated type, with `STRING` as the underlying storage type and UUID as the value.
TYPE UserID UUID

# Now, unlike the `ProtoUser` above, `User`-s have IDs.
TYPE User {
  uid UserID
  name STRING
  age INT
}
```

Also, the DB record:

```
# This one-liner is legal.
DB { users TABLE<User> { PRIMARY_KEY uid } }

# NOTE(dkorolev): I'm still thinking if this syntax is best.
```

## Implement business logic in the `COMMAND`

This also demonstrates that `COMMAND`-s can have custom return types. This example does not have HTTP in mind as the only and/or recommended calling convention transport, althouth, of course, it is a one-liner to expose this command as an HTTP one.

Types:

```
TYPE PutUserResponseStatus ENUM INTEGER {
  OK = 0
  ALREADY_EXISTS = 409
}

TYPE PutUserResponse {
  status PutUserResponseStatus 
  message STRING
}
```

The command:

```
COMMAND PutUserIfNotPresent(u User) RETURNS PutUserResponse {
  IF NOT users HAS_KEY u.uid {
    PUT users u
    RETURN PutUserResponse{PutUserResponseStatus.OK, "User added."}
  } ELSE {
    RETURN PutUserResponse{PutUserResponseStatus.ALREADY_EXISTS, "The user with this ID already exists."}
  }
}
```

Alternatively:

```
COMMAND PutUserIfNotPresent(u User) RETURNS PutUserResponse {
  if PUT_IF_ABSENT users u {
    RETURN PutUserResponse{PutUserResponseStatus.OK, "User added."}
  } ELSE {
    RETURN PutUserResponse{PutUserResponseStatus.ALREADY_EXISTS, "The user with this ID already exists."}
  }
}
```

## Add support for `.email`

Add email addresses to users. And respect the constraint that no two users can have the same email.

We could use users' emails as keys, but this would be too easy. Instead, let's add another table ("dictionary") that map registered email addresses to registered user IDs.

```
TYPE EmailToUserMapping {
  email STRING
  uid UserID
}
```

```
DB { user_emails TABLE<EmailToUserMapping> { PRIMARY_KEY email } }
```

```
TYPE PutUserResponseStatus ENUM INTEGER {
  OK = 0
  EMAIL_ALREADY_REGISTERED = 1
  ALREADY_EXISTS = 2
}

TYPE PutUserResponse {
  status PutUserResponseStatus 
  message STRING
  user_id NULLABLE<STRING>
  email NULLABLE<STRING>
}
```

```
COMMAND PutUserIfNotPresent(u User) RETURNS PutUserResponse {
  IF NOT users HAS_KEY users u.uid {
    IF NOT user_emails HAS_KEY u.email {
      PUT users u
      PUT user_emails { email: u.email, user_id: u.uid }
      RETURN { OK, "User added." }
    } ELSE {
      RETURN {
        EMAIL_ALREADY_REGISTERED     # A positional argument.
        "Email already registered."  # Also a positional argument.
        email: u.email               # A named argument.
                                     # Note that `user_id` is missing.
      }
    }
  } ELSE {
    RETURN {
      USER_ALREADY_EXISTS
      "User already exists."
      user_id: u.id
    }
  }
}
```

## Add `email` validation

First, extend the storage to include email verification codes and the necessary timestamps.

```
TYPE EmailValidationCode {
  secret_code STRING
  sent_time EPOCH_MS
  expiration_time EPOCH_MS
}

TYPE EmailToUserMapping {
  email STRING
  uid UserID

  validation_code OPTIONAL<EmailValidationCode>
}

DB { user_emails TABLE<EmailToUserMapping> { PRIMARY_KEY email } }
```

Also, extend the `User` type and add the `email_validated` field.

```
TYPE User {
  uid UserID
  name STRING
  age INT
  email_validated BOOLEAN DEFAULT FALSE
}
```

Now, the command and its schema.

```
TYPE ValidateEmailRequest {
  email STRING
  secret_code STRING
  uid UserID
}

TYPE ValidateEmailResponseCode ENUM INTEGER {
  OK = 0
  ERROR_VALIDATING_EMAIL = 1
}

TYPE ValidateEmailResponse {
  email STRING
  uid UserUD
  code ValidateEmailResponseCode
}
```

```
COMMAND ValidateEmail(v ValidateEmailRequest) RETURNS ValidateEmailResponse {
  IF NOT user_emails HAS_KEY v.email {
    RETURN {
      ERROR_VALIDATING_EMAIL  # Do not share that the email does not exist.
      email: v.email
    }
  }

  email_record = GET user_emails v.email
  u = GET users (GET user_emails v.email).uid

  IF u != v.uid {
    RETURN {
      ERROR_VALIDATING_EMAIL  # Do not share that the email belongs to a different user.
      email: v.email
    }
  }

  IF email_record.code u != v.uid {
    RETURN {
      ERROR_VALIDATING_EMAIL  # Do not share that the email belongs to a different user.
      email: v.email
    }
  }

  IF NOT EXISTS email_record.validation_code {
    RETURN {
      ERROR_VALIDATING_EMAIL  # Do not share that the validation code was not sent to this email.
      email: v.email
    }
  }
  validation_code = VALUE email_record.validation_code

  IF u.email_validated {
    # NOTE(dkorolev): A future unit test: if the user changes their email, need to re-validate it.
    RETURN {
      OK  # Do not share that the user's email was already validated before this call.
      email: v.email
    }
  }

  IF (v.secret_code == validation_code.secret_code) AND
     (NOW_MS() < validation_code.expiration_time) {
    updated_u = EXTEND u { email_validated: true }
    PUT users updated_u
    updated_email_record = OMIT email_record { validation_code }
    PUT user_emails updated_email_record
    RETURN {
      OK
      email: v.email
    }
  } ELSE {
    RETURN {
      ERROR_VALIDATING_EMAIL  # Do not share why did the verification process fail.
      email: v.email
    }
  }
}
```

## Support the logic to send the validation code.

```
TYPE GenerateEmailValidationCodeResponse ENUM INTEGER {
  OK = 0
  USER_DOES_NOT_EXIST = 1
  TRY_LATER = 2
  INTERNAL_ERROR = -1
}

COMMAND SendEmailValicationCode(uid UserID, code STRING) RETURNS GenerateEmailValidationCodeResponse {
  IF NOT users HAS_KEY uid {
    RETURN { USER_DOES_NOT_EXIST }
  }
  email = (users GET users uid).email
  IF NOT user_emails HAS_KEY email {
    RETURN { INTERNAL_ERROR }  # Should not have a user without an email.
  }
  rec = user_emails GET email
  IF (NOT EXISTS rec.validation_code) OR
     (VALUE(rec.validation_code).sent_time > NOW_MS() - kEmailCooldownMinutes * 60 * 1000) {
    RETURN { TRY_LATER }
  }
  user_emails PUT (EXTEND rec {
    validation_code: {
      secret_code: code
      send_time: NOW_MS()
      expiration_time = NOW_MS() + kEmailCodeValidityPeriodMinutes * 60 * 1000
    }
  })
  RETURN { OK }
}
```

## TODO

* Make user ID autogeneratable. This enables POST.
* Make it clear that the syntax allows replacing commas by newlines.
* Consider `:=` instead of `=`.
