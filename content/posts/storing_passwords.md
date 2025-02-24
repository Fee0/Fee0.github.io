+++
title = "How to store passwords"
date = "2025-02-15"
keywords = ["password","store","encryption","hash","salt","bcrypt","sso"]

[extra]
repo_view = false
comment = false
+++

# Why store passwords?
Most Apps or services require some form of a user authentication. Knowing which individual user the service is currently talking to provides many benefits. E.g. to store information about the user, store settings or protect the service with rate limits. However, storing passwords carelessly can lead to serious security breaches, exposing user data and even compromise additional services as users tend to use the same password for many services.

# Plaintext
The simplest approach to storing passwords is to store them in plain text. 

<center>

| User ID | Username | Password             | 
|---------|----------|----------------------|
| 1       | Alice    | password123          | 
| 2       | Bob      | qwerty               | 
| 3       | Charlie  | 12345678             | 

</center>

A user enters his username and the password and the server compares both against the entries in its database. This might be secure as long as no adversary can ever access the database. But, any malicious access to the database results in all passwords being immediately visible. This is the least secure way to store passwords. Let’s try something better.

# Encrypted
Encrypting the password can hide it from an adversary. 

<center>

| User ID | Username | Encrypted Password             | 
|---------|----------|----------------------|
| 1       | Alice    | 7c97...          | 
| 2       | Bob      | c881...               | 
| 3       | Charlie  | 4ad4...             | 

</center>

However, they key to decrypt the passwords must be available somewhere as it is required for each login. It’s not unlikely that an adversary can access the key too which makes it trivial to show all passwords in plaintext again. The key and ability to to decrypt are problematic in this scenario.

# Hashing
While encryption is a two-way function (encryption and decryption), we don't really need or want the ability to decrypt as we could compared the encrypted passwords and still find a matching password like that. Instead, a [hash function](https://en.wikipedia.org/wiki/Hash_function) is designed as a one-way function that is not reversible and therefore does not necessarly need a key either. Now we can simply calculate the digest and store it instead of the password:

<center>

SHA2("password123") = EF92B778BAFE771E89245B89ECBC08A44A4E166C06659911881F383D4473E94F
</center>

Here we are using [SHA-2](https://en.wikipedia.org/wiki/SHA-2) as a hash function for every password.

<center>

| User ID | Username | Password Hash             | 
|---------|----------|----------------------|
| 1       | Alice    | EF92...          | 
| 2       | Bob      | 65E8...               | 
| 3       | Charlie  | EF79...             | 

</center>

Thanks to the hash function, there is no easy way back from the message digest to the plain password, except for trying all possible password combinations, hashing them, and comparing the resulting digests.

However, we still have a human factor: the strength of the password. An adversary could pre generate large tables ([rainbow tables](https://de.wikipedia.org/wiki/Rainbow_Table)) of password-hash-combinations for all shorter passwords or for all common passwords. 

<center>

| Plaintext     | Hash      |
|--------------|--------------------------------------|
| a            | CA97...  |
| b            | 3E23...  |
| c            | A3A5...  |
| ..           | ...      |
| password123  | EF92...  |
| ...          | ...      |

</center>

All the attacker has to do now is to compare the retrieved hashes against the precomputed hashes in the rainbow tables. Simple passwords can quickly be found like this.

# Salting
Instead of simply hashing the user password, we could prepend a so called random [salt](https://en.wikipedia.org/wiki/Salt_%28cryptography%29) to each password before hashing:

<center>

SHA2("1234" + "password123") = B08EB873685F5DC960FF6229AEA2096BBE82EDD0DEC877D3E94DD3A3491BBB03

</center>

Then we store the salt next to the password in the database. 

<center>

| User ID | Username | Salt    | Password Hash   |
|---------|----------|---------|----------------|
| 1       | Alice    | 512ba9  | B08E...        |
| 2       | Bob      | 03fc72  | 65E8...        |
| 3       | Charlie  | 825ef0  | EF79...        |

</center>

Whenever we get a new password from a login, we prepend the salt, compute the hash, and compare the digests. The salt should be randomly chosen for each user. Then, precomputing rainbow tables becomes inefficient as the salt makes the passwords longer which results in gigantic tables that require terra bytes of storage.

# Slow Hash
However, even without precomputing hashes, an adversary could still try many passwords, prepend the salt and check for a match. Through faster and faster hardware this attack can still find many passwords rather efficiently even without precomputed rainbow tables.

In order to stop these attacks, the hash processing needs to be forced to slow down. This is why some modern hash functions have parameters to enforce longer computation times. For example, [bcrypt](https://en.wikipedia.org/wiki/Bcrypt) takes as input an additional cost parameter that increases the cost of computation. The output includes a version identifier, the cost, the salt and digest in this format:

<center>

\\$[version]\\$[cost]\\$[22 character salt][31 character hash]

</center>

This digest we can store in the database. Depending on the cost the computation for one hash could take multiple seconds which significantly decreases the amount of passwords that can be guessed by the adversary given the same amount of time.

<center>

| User ID | Username | Bcrypt Digest            
|---------|----------|------------------------------------------|
| 1       | Alice    |  \\$2a\\$15\\$Yyeoa/khr3.gIRkkOJdH6.arOyiGR4eklXI1d9acSDMGriAGMaaGu          | 
| 2       | Bob      |  \\$2a\\$15\\$NJFb8moHVvRuItbxxlm31eYNPhGYd/o2hWgCtyDIzW80Kn1LNMGgq              | 
| 3       | Charlie  |  \\$2a\\$15\\$32zu6pIGsvNRQ5I7QJpf4OIFVlTgE2riKBDAhwISt6x3GT9NWFAhC            | 

</center>

# No password
Even though we made it very difficult for an adversary to get to the user passwords, they can still start to guess passwords. So, instead of storing passwords at all for a service it might useful to let authentication be handled by a trusted identity provider. Authentication is no longer handled by the service itself but by some provider like Google or Microsoft which store the passwords and allow you to log in into many different services with one [Single Sign-On (SSO)](https://en.wikipedia.org/wiki/Single_sign-on).


