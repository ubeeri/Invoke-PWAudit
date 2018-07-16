# Invoke-PWAudit

Invoke-PWAudit is designed to discover similarly named accounts with shared passwords in Windows Active Directory. It works by DCSyncing password hashes for given accounts, and then either comparing hashes with other accounts, or using the hashes to test authentication against other accounts. It will search for "matching" usernames given prefixes and suffixes, as well as direct matches.

There are many scenarios where Invoke-PWAudit can be used, which are outlined in this readme.

**The tool requires Domain Administrator (or equivalent) permissions in at least one domain to function.**

## Comparing users within a single Windows Domain

To compare users' passwords within a single domain, you will need to specify possible prefixes or appendices that the tool will search for (since there cannot be users with identical usernames in one domain). This is done with the `-Prepend` and `-Append` options.

### A simple suffix (john and john_a)

For example, say you have a group of "standard users" (john, george, and fred) who have privileged accounts designated with a "\_a" (john_a, george_a, and fred_a), all in the domain "local.corp". You could use the following command to compare passwords between these users' standard and privileged accounts:

```
Invoke-PWAudit -Domain1 local.corp -Append "_a"
```

If john and john_a had the same password, the output would look something like this:

```
Found 1 user sets with matching passwords:
john
john_a
```

Note that we explicitly specify the domain here with Domain1. This will default to the current domain if not set.

### Multiple prefixes and suffixes

The tool can handle multiple prefixes and suffixes simultaneously. Let’s say your account naming scheme was \[department letter\]\[user id\]\[privilege level\]. Pretend we had a user with an unprivileged (T4325U) and privileged (T4325A) tech support account, as well as an privileged server admin account (S4325A). Now we need to specify 2 prefixes and 2 suffixes to catch all these usernames in our search:

```
Invoke-PWAudit -Append U,A -Prepend T,S
```

Let’s say just T4325U and S4325A share a password. The tool will identify all 3 of these matching accounts:

```
Found 1 matching username sets:
T4325U
T4325A
S4325A

DCSync all users and compare password hashes? (Y/n)
```
If we continue, just the accounts with matching passwords will be returned:

```
Found 1 user sets with matching passwords:
T4325U
S4325A
```

## Comparing users across Windows Domains

To compare users across domains, simply specify both a `-Domain1` and `-Domain2` parameter. (`-Domain1` will default to the current domain if not specified, so you may not need it.)

### Exact matches

If just the domains are specified, the tool will search for usernames that match exactly (case-insensitive) between the domains and compare those. 
```
Invoke-PWAudit -Domain1 domain.one -Domain2 domain.two
```
This would find and compare user sets such as domain.one\john & domain.two\john.

### Append and Prepend across domains

`-Append` and `-Prepend` can be used here as well, but are not required as usernames can have exact matches between domains. If you do use these modifiers they will **only apply to usernames in Domain2**. For example:
```
Invoke-PWAudit -Append "_a" -Domain1 domain.one -Domain2 domain.two
```
would find a set of users such as domain.one\john, domain.two\john, and domain.two\john_a. It would **not** find domain.one\john_a however.

### Testing auth in the 2nd domain

If you do not have permissions to DCSync users in a second domain, you can use the `-TestAuth` argument to actually attempt to authenticate users in that domain using the hashes gathered via DCSync in the first domain.

**This will cause authentication attempts against accounts that are found, so you risk locking out accounts if it is run multiple times, or if an account already only has one attempt left.**
```
Invoke-PWAudit -Domain1 domain.one -Domain2 domain.two -TestAuth
```
Let's assume that the tool found the set of users domain.one\john & domain.two\john. With `-TestAuth` specified, the tool will DCSync the password hash for domain.one\john, then use that hash to attempt to authenticate as domain.two\john to domain.two.

(Authentication is tested by establishing an SMB connection to \\\\domain.two\\netlogon\\)

## Using a .csv input file

If you already know what users you want to compare, you can put them in a csv\* file and pass that to the tool.
\* It's not really a proper csv file, there are no headers, and lines can have varying numbers of elements.

Each line is simply a comma delimited list of users to test against each other. A domain can be specified, or the tool will use the value of the `-Domain1` argument (defaults to the current domain if also not specified) where only a username is given.

By using a .csv file, you can compare dis-similar usernames, as well as usernames across more than two domains at once.

A sample file could look like this:
```
george,george_a
domain.one\john,domain.two\john,domain.two\john_a
domain.two\fred,domain.three\henrey
```
If this were saved as users.csv, it would be passed to the tool using `-CSV` as such:
```
Invoke-PWAudit -CSV users.csv
```
Since we didn't specify a `-Domain1` here, the users without a domain given (george & george_a) would default to the current domain where the tool was run from.

### Testing auth with a .csv file

If `-TestAuth` is specified when using a .csv file, the first user of every line will be DCSynced, and then their password hash used to attempt authentication with the other users on the same line. If the same .csv as above were used
```
Invoke-PWAudit -CSV users.csv -TestAuth
```
[current.domain]\george, domain.one\john and domain.two\fred would all be DCSynced. The password hashes returned would then be used to authenticate as the other users on each line respectively.

## Whitelisting and blacklisting users

You can specify to only search for certain users, or conversely, specify to skip certain users. If in the 2nd example, we only wanted to look at user 4325, we could specify that with `-Users T4325U`. (Any of 4325's 3 usernames could be used here, and the other 2 would be found.)

The option for skipping users works the same way. If we didn't want to touch george or fred's accounts, we could specify `-Skip george,fred`

## Running without prompts

By default, the tool will prompt before actually DCSyncing or testing authentication against any users that it discovered through searching. To skip this, you can specify `-NoConfirm`. This does not apply when using a .csv file input, as all of the usernames are already user chosen.

