---
title: Upgrade to Kimai 2 from v1
description: Install Kimai 2 and import your existing timesheet data from Kimai 1
canonical: /documentation/migration-v1.html
---

This documentation covers all necessary steps to migrate from Kimai 1 to Kimai 2.

{% alert %}
You can <a href="{% link _pages/support.html %}">get professional support</a> if you are not sure about performing the upgrade yourself. 
{% endalert %}

## Introduction

Before starting with the migration, please read the following FAQs:

- Data from the existing v1 installation is only read and will never be changed
- Data can only be imported from a Kimai installation with at least `v1.0.1` and database revision `1388` (check your `configuration` table)
- User-specific rates are not yet supported in Kimai 2, but
  - fixed-rates and hourly-rates for projects and activities are imported
  - fixed-rates and hourly-rates and total rate for timesheet entries are imported
- Customers in Kimai 2
  - are only used for recording, they cannot login and no user accounts will be created for them
  - have a country code, which can be set during import or edited afterwards (Kimai v1 doesn't know the country)
  - have a currency code, which can be set during import or edited afterwards (Kimai v1 only knows one global currency)
- You have to supply a default password, which will be used for every imported user(!)
  - due to security issues we cannot import the original passwords
  - let your users reset it afterwards with the [Password reset]({% link _documentation/users.md %}) function
- The import will fail, if a user from v1 has either an empty email OR the same email is used for multiple users
- Data which was deleted in Kimai v1 (user, customer, projects, activities) will be imported and set to `invisible`
  - if you don't want that, you have to delete all entries that have the value `1` in the `trash` column before importing
- Groups import
  - The import will skip groups that have no members. 
  - The import will always assign a teamlead to the project. If none of the users in Kimai v1 was assigned as the teamlead, the first member of the group is assigned as teamlead during import. 
  - The groups that were trashed in Kimai v1 are not imported into Kimai 2 as hidden/trashed teams are not supported.

## Install Kimai

You have to install Kimai, please read the [documentation]({% link _documentation/installation.md %}) first.
You can install it on the same server, but remember that you have to meet the server requirements (see [downloads page]({% link _pages/download.md %})).

After Kimai 2 runs properly, the actual *migration* takes place, by importing the data from your Kimai 1 database into Kimai 2.
You have to have SSH access to your server, as you will use a command shipped with Kimai 2, which will pull the data into the configured database from your `.env` file.

The database does not have to be on the same server and the database user (for the Kimai 1 tables) needs only read access.
     
## Database import

{% alert warning %}It is strongly recommended to test the import, as unexpected problems may occur. If you already created data (like users and customers), backup your Kimai 2 database before performing the first tests!{% endalert %} 

See the help for the import command and all its arguments by executing:

```bash
bin/console kimai:import --help
```

A full command could look like this:
```bash
bin/console kimai:import-v1 --global --timezone="timezone" --language="language" "mysql://user:password@127.0.0.1:3306/database?charset=utf8" "db_prefix" "password" "country" "currency" 
```

The fields `country` and `currency` are optional and will be set to DE and EUR if not given.
The flags `timezone` and `language` are used for imported users, in case they didn't set it in Kimai 1 (you should update the preference in Kimai 1 before importing, if you work across different timezones).

It is recommended to test the import in a fresh database. You can test your import as often as you like and fix possible problems in your installation.
A sample command could look like that:
```bash
bin/console doctrine:schema:drop --force && \
bin/console kimai:install -n && \
bin/console kimai:import-v1 --global --timezone="Europe/Zurich" --language="ch" "mysql://kimai:test@127.0.0.1:3306/kimai?charset=latin1" "kimai_" "NEW-PASSWORD-1234" "CH" "CHF"
```
That will drop the configured Kimai database schema and re-create it, before importing the data from the `mysql` database at `127.0.0.1` on port `3306` authenticating the user `kimai` with the password `test` for import.
The connection will use the charset `latin1` and the default table prefix `kimai_` for reading data. Imported users can login with the password `test123` and all customer will have the country `CH` and the currency `CHF` assigned.

### Problems and solution
 
Kimai 1 was written a long time ago, when MySQL was lacking proper UTF8 support and foreign keys (in shared hostings).
While [migrating dozens of customers installations]({% link _pages/support.html %}) I stumbled upon some recurring problems, 
that can be solved with some SQL commands.

{% alert warning %}Be aware, depending on your Kimai 1 version the field names might be different in the following snippets{% endalert %} 

#### Broken character

Many Kimai 1 installations have broken special character (like german umlauts or other language specific non-ascii characters) in the database.

This problem does not show up in the frontend og Kimai 1, as the database connection is using a different collation then the database. 
But you can see these problems, when you query the database directly (eg. with a tool like phpMyAdmin). 

You can find these broken entries (mainly timesheet descriptions) with SQL statements like these (in the Kimai 1 database):
```sql 
SELECT * FROM `kimai_timeSheet` WHERE comment like "%Ã¤%";
SELECT * FROM `kimai_timeSheet` WHERE comment like "%Ã„%";
SELECT * FROM `kimai_timeSheet` WHERE comment like "%Ã¼%";
SELECT * FROM `kimai_timeSheet` WHERE comment like "%Ãœ%";
SELECT * FROM `kimai_timeSheet` WHERE comment like "%Ã¶%";
SELECT * FROM `kimai_timeSheet` WHERE comment like "%Ã–%";
SELECT * FROM `kimai_timeSheet` WHERE comment like "%ÃŸ%";
```

They are not fixed automatically by Kimai, but changing them is then just a matter of rewriting these SQL queries:
```sql 
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "Ã¤", "ä") WHERE comment like "%Ã¤%";
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "Ã„", "Ä") WHERE comment like "%Ã„%";
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "Ã¼", "ü") WHERE comment like "%Ã¼%";
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "Ãœ", "Ü") WHERE comment like "%Ãœ%";
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "Ã¶", "ö") WHERE comment like "%Ã¶%";
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "Ã–", "Ö") WHERE comment like "%Ã–%";
UPDATE `kimai_timeSheet` SET comment = REPLACE(comment, "ÃŸ", "ß") WHERE comment like "%ÃŸ%";

UPDATE `kimai_timeSheet` SET description = REPLACE(description, "Ã¤", "ä") WHERE description like "%Ã¤%";
UPDATE `kimai_timeSheet` SET description = REPLACE(description, "Ã„", "Ä") WHERE description like "%Ã„%";
UPDATE `kimai_timeSheet` SET description = REPLACE(description, "Ã¼", "ü") WHERE description like "%Ã¼%";
UPDATE `kimai_timeSheet` SET description = REPLACE(description, "Ãœ", "Ü") WHERE description like "%Ãœ%";
UPDATE `kimai_timeSheet` SET description = REPLACE(description, "Ã¶", "ö") WHERE description like "%Ã¶%";
UPDATE `kimai_timeSheet` SET description = REPLACE(description, "Ã–", "Ö") WHERE description like "%Ã–%";
UPDATE `kimai_timeSheet` SET description = REPLACE(description, "ÃŸ", "ß") WHERE description like "%ÃŸ%";

UPDATE `kimai_users` SET name = REPLACE(name, "Ã¤", "ä") WHERE name like "%Ã¤%";
UPDATE `kimai_users` SET name = REPLACE(name, "Ã„", "Ä") WHERE name like "%Ã„%";
UPDATE `kimai_users` SET name = REPLACE(name, "Ã¼", "ü") WHERE name like "%Ã¼%";
UPDATE `kimai_users` SET name = REPLACE(name, "Ãœ", "Ü") WHERE name like "%Ãœ%";
UPDATE `kimai_users` SET name = REPLACE(name, "Ã¶", "ö") WHERE name like "%Ã¶%";
UPDATE `kimai_users` SET name = REPLACE(name, "Ã–", "Ö") WHERE name like "%Ã–%";
UPDATE `kimai_users` SET name = REPLACE(name, "ÃŸ", "ß") WHERE name like "%ÃŸ%";

UPDATE `kimai_activities` SET name = REPLACE(name, "Ã¤", "ä") WHERE name like "%Ã¤%";
UPDATE `kimai_activities` SET name = REPLACE(name, "Ã„", "Ä") WHERE name like "%Ã„%";
UPDATE `kimai_activities` SET name = REPLACE(name, "Ã¼", "ü") WHERE name like "%Ã¼%";
UPDATE `kimai_activities` SET name = REPLACE(name, "Ãœ", "Ü") WHERE name like "%Ãœ%";
UPDATE `kimai_activities` SET name = REPLACE(name, "Ã¶", "ö") WHERE name like "%Ã¶%";
UPDATE `kimai_activities` SET name = REPLACE(name, "Ã–", "Ö") WHERE name like "%Ã–%";
UPDATE `kimai_activities` SET name = REPLACE(name, "ÃŸ", "ß") WHERE name like "%ÃŸ%";

UPDATE `kimai_projects` SET name = REPLACE(name, "Ã¤", "ä") WHERE name like "%Ã¤%";
UPDATE `kimai_projects` SET name = REPLACE(name, "Ã„", "Ä") WHERE name like "%Ã„%";
UPDATE `kimai_projects` SET name = REPLACE(name, "Ã¼", "ü") WHERE name like "%Ã¼%";
UPDATE `kimai_projects` SET name = REPLACE(name, "Ãœ", "Ü") WHERE name like "%Ãœ%";
UPDATE `kimai_projects` SET name = REPLACE(name, "Ã¶", "ö") WHERE name like "%Ã¶%";
UPDATE `kimai_projects` SET name = REPLACE(name, "Ã–", "Ö") WHERE name like "%Ã–%";
UPDATE `kimai_projects` SET name = REPLACE(name, "ÃŸ", "ß") WHERE name like "%ÃŸ%";

UPDATE `kimai_projects` SET comment = REPLACE(comment, "Ã¤", "ä") WHERE comment like "%Ã¤%";
UPDATE `kimai_projects` SET comment = REPLACE(comment, "Ã„", "Ä") WHERE comment like "%Ã„%";
UPDATE `kimai_projects` SET comment = REPLACE(comment, "Ã¼", "ü") WHERE comment like "%Ã¼%";
UPDATE `kimai_projects` SET comment = REPLACE(comment, "Ãœ", "Ü") WHERE comment like "%Ãœ%";
UPDATE `kimai_projects` SET comment = REPLACE(comment, "Ã¶", "ö") WHERE comment like "%Ã¶%";
UPDATE `kimai_projects` SET comment = REPLACE(comment, "Ã–", "Ö") WHERE comment like "%Ã–%";
UPDATE `kimai_projects` SET comment = REPLACE(comment, "ÃŸ", "ß") WHERE comment like "%ÃŸ%";

UPDATE `kimai_customers` SET name = REPLACE(name, "Ã¤", "ä") WHERE name like "%Ã¤%";
UPDATE `kimai_customers` SET name = REPLACE(name, "Ã„", "Ä") WHERE name like "%Ã„%";
UPDATE `kimai_customers` SET name = REPLACE(name, "Ã¼", "ü") WHERE name like "%Ã¼%";
UPDATE `kimai_customers` SET name = REPLACE(name, "Ãœ", "Ü") WHERE name like "%Ãœ%";
UPDATE `kimai_customers` SET name = REPLACE(name, "Ã¶", "ö") WHERE name like "%Ã¶%";
UPDATE `kimai_customers` SET name = REPLACE(name, "Ã–", "Ö") WHERE name like "%Ã–%";
UPDATE `kimai_customers` SET name = REPLACE(name, "ÃŸ", "ß") WHERE name like "%ÃŸ%";

UPDATE `kimai_expenses` SET designation = REPLACE(designation, "Ã¤", "ä") WHERE designation like "%Ã¤%";
UPDATE `kimai_expenses` SET designation = REPLACE(designation, "Ã„", "Ä") WHERE designation like "%Ã„%";
UPDATE `kimai_expenses` SET designation = REPLACE(designation, "Ã¼", "ü") WHERE designation like "%Ã¼%";
UPDATE `kimai_expenses` SET designation = REPLACE(designation, "Ãœ", "Ü") WHERE designation like "%Ãœ%";
UPDATE `kimai_expenses` SET designation = REPLACE(designation, "Ã¶", "ö") WHERE designation like "%Ã¶%";
UPDATE `kimai_expenses` SET designation = REPLACE(designation, "Ã–", "Ö") WHERE designation like "%Ã–%";
UPDATE `kimai_expenses` SET designation = REPLACE(designation, "ÃŸ", "ß") WHERE designation like "%ÃŸ%";

UPDATE `kimai_expenses` SET comment = REPLACE(comment, "Ã¤", "ä") WHERE comment like "%Ã¤%";
UPDATE `kimai_expenses` SET comment = REPLACE(comment, "Ã„", "Ä") WHERE comment like "%Ã„%";
UPDATE `kimai_expenses` SET comment = REPLACE(comment, "Ã¼", "ü") WHERE comment like "%Ã¼%";
UPDATE `kimai_expenses` SET comment = REPLACE(comment, "Ãœ", "Ü") WHERE comment like "%Ãœ%";
UPDATE `kimai_expenses` SET comment = REPLACE(comment, "Ã¶", "ö") WHERE comment like "%Ã¶%";
UPDATE `kimai_expenses` SET comment = REPLACE(comment, "Ã–", "Ö") WHERE comment like "%Ã–%";
UPDATE `kimai_expenses` SET comment = REPLACE(comment, "ÃŸ", "ß") WHERE comment like "%ÃŸ%";
```

#### User accounts without email

Find and update all users, that have no email address: 
```sql
SELECT * FROM `kimai_users` WHERE mail = '' OR mail IS NULL;
UPDATE `kimai_users` SET mail = concat(name, '@example.com') WHERE mail = '' OR mail IS NULL;
```

#### Reset password for testing

Update an account with a new `password` in your Kimai 1 database:
```sql
UPDATE `kimai_users` SET password = md5(concat('your-salt', 'new-password', 'your-salt')) WHERE userID = XYZ;
```
