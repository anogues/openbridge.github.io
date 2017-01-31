# Documentation · Data Pipeline

You're looking at the docs for the Openbridge Data Pipeline product! The Pipeline product allows non-technical users a simple and automated toolset to deliver, process and store data of all sizes to a private warehouse. Let's dive in.

- [What Is A Data Pipeline?](#what-is-a-data-pipeline)
- [Files to Database](#files-to-database)
	- [Getting Organized](#getting-organized)
		- [Example: CRM Files](#example-crm-files)
	- [Understanding Your File Layouts](#understanding-your-file-layouts)
	- [File Naming](#file-naming)
	- [File Structure/Layout](#file-structurelayout)
	- [Dealing With File Layouts Changes](#dealing-with-file-layouts-changes)
	- [Directories](#directories)
- [Sending Compressed Files](#sending-compressed-files)
	- [Simple Use Case: One Archive, One File](#simple-use-case-one-archive-one-file)
	- [Complex Use Case: One Archive, Many Files](#complex-use-case-one-archive-many-files)
- [Encoding Your Files](#encoding-your-files)
	- [Check Encoding](#check-encoding)
- [How To Deliver Data](#how-to-deliver-data)
	- [Transfer Protocols](#transfer-protocols)
		- [Secure File Transfer Protocol (SFTP)](#secure-file-transfer-protocol-sftp)
		- [FTP Explicit Mode (TLS)/(SSL)](#ftp-explicit-mode-tlsssl)
		- [File Transfer Protocol (FTP)](#file-transfer-protocol-ftp)
	- [Check Your Firewall](#check-your-firewall)
	- [Blocked Files](#blocked-files)
	- [Hidden Files](#hidden-files)
- [Error Handling](#error-handling)
	- [Bulk Transfers](#bulk-transfers)
		- [Manifests](#manifests)
	- [Status Codes](#status-codes)
	- [File Integrity](#file-integrity)
- [Connection Considerations](#connection-considerations)
	- [DNS-based Blackhole List (DNSBL) / Real-time Blackhole List (RBL)](#dns-based-blackhole-list-dnsbl-real-time-blackhole-list-rbl)
	- [Anti-Virus, Malware and Trojans](#anti-virus-malware-and-trojans)
	- [Account Ban and Lockout](#account-ban-and-lockout)
	- [Idle Connection Time Limits](#idle-connection-time-limits)
- [Reference](#reference)
	- [FTP Clients](#ftp-clients)
	- [GUI](#gui)
		- [Free](#free)
		- [Paid](#paid)
	- [CLI](#cli)
	- [Python](#python)
- [FAQs](#faqs)

<!-- /TOC -->


# What Is A Data Pipeline?

A data pipeline is a series of well designed, intuitive, and cohesive components--built on top of he Openbridge platform. If you are familar with loading data to relational data stores you'll find this familiar, but simpler and more automated.

An important concept for data pipelines is understanding how your data is delivered and organized. Typically, you will want to make sure that the relationship between the file(s) being delivered align with how you want to have them stored in the database. Not only does this insure accuracy and consistency, it is a key element to the "hands-off" automation of your data pipeline workflows.

# Files to Database

## Getting Organized

We will start with a simple structure that reflects different types of data. First, we have a parent `customers` directory. Within `customers`, there are four subdirectories for `address`, `offers`, `rewards` and `transactions`. The parent and subdirectories reflect a structure that aligns with how data needs to be organized for delivery by external systems.

```
customers/
  ├── address/
  ├── offers/
  ├── rewards/
  └── transactions/
```

With the structure in place, data can be delivered. However, it is important to understand that there needs to be some organizing principles on what data is delivered and where it is delivered. For Openbridge to properly process and route your data, each pipeline needs to have a definition that maps to the object and/or files being delivered.

### Example: CRM Files

For example, based on the `customers` directory structure we defined previously, a collection of `csv` and `zip` files have been delivered into these locations.

```
    customers/
      ├── 2016-01-01_customers.csv
      ├── 2016-01-02_customers.csv
      ├── 2016-01-03_customers.csv
      ├── address/
      │   ├── address_dma_lookup/
      │   ├── address_weather_lookup/
      │   ├── 2016-01-01_address.csv
      │   └── 2016-01-02_address.csv
      ├── offers/
      │   ├── offers_open/
      │   └── offers_targeted/
      │       ├── 2016-01-01_offers.zip
      │       ├── 2016-01-02_offers.zip
      ├── rewards/
      │   ├── 2016-01-01_rewards.zip
      │   ├── 2016-01-02_rewards.zip
      │   ├── 2016-01-03_rewards.zip
      │   ├── 2016-01-04_rewards.zip
      │   └── 2016-01-05_rewards.zip
      └── transactions/
          ├── 2016-01-01_transactions.csv
          ├── 2016-01-02_transactions.csv
          ├── 2016-01-03_transactions.csv
          ├── 2016-01-04_transactions.csv
          ├── 2016-01-05_transactions.csv
          └── 2016-01-06_transactions.csv
```

The simplest use case is that a table is generated according to the location a file is delivered. In the above example you will have a `customers`, `address`, `offers_targeted` and `transactions` tables. The content of each table will be based on the corresponding `.csv` or `zip` files placed in each directory.

## Understanding Your File Layouts

Please note, that each `.csv` within each directory must share the same data structure/layout. For example, all the `"*_transactions.csv"` files located in the `customers/transactions/` directory share the following structure:

```
    "ID","DATE","CUSTOMERID","PURCHASE","STOREID"
    "0122","December 10, 2015","123432100A","8.43","4985"
    "0123","December 10, 2015","543421111A","2.61","3212"
```

When additional `"*_transactions.csv"` files are delivered to `customers/transactions/` they will be loaded to a table named `transactions` that looks like this...


|**id**|**date**|**customerid**|**purchase**|**storeid**|
|---|---|---|---|---|
|0122|December 10, 2015|123432100A|8.43|4985|
|0123|December 10, 2015|543421111A|2.61|3212|
|...|...|...|...|...|


## File Naming

The platform accepts any UTF-8 encoded delimited text format (e.g. comma, tab, pipe, etc.) with column headers (header row) and a valid file type extension (`.csv` or `.txt`).

File names should meet the 3 criteria outlined below.

- **Descriptive**: The file name should include the data source, data description and date/date range associated with the data in each file (e.g. `socialcom_paidsocial_20140801.txt` or `socialcom_paidsocial_20140701_20140801.txt`)
- **Unique**: Unique files will be stored and accessible for auditing purposes. Files posted with the same name will be overwritten, making auditing impossible (e.g. `socialcom_paidsocial_20140801.txt`, `socialcom_paidsocial_20140901.txt`, `socialcom_paidsocial_20141001.txt`
- **Consistent**: File naming patterns should be kept consistent over time to enable automated auditing (e.g. `socialcom_paidsocial_20140801.txt`, `socialcom_paidsocial_20140901.txt`, `socialcom_paidsocial_20141001.txt`

This will facilitate any required auditing/QC of data received. **It is highly likely that use of non-standard file naming conventions will cause your import pipeline to fail or result in an outcome different than expected.**

## File Structure/Layout

The file column headers are important as they will also be the field names used in the corresponding table. As with the folder name, the column headers should be descriptive enough for the end user to understand its contents. The column headers must meet the same syntax requirements. These requirements are outlined below...

- Must start with a letter
- Must contain only lowercase letters
- Can contain numbers and underscores `('_')` but no spaces or special characters (e.g. `'@'`, `'#'`, `'%'`)

Valid column header:

```
    "inquiries_to_purchase_pct"
```

Invalid column header:

```
    "Inquiries to Purchase %"
```

If the field names in the batch file do not meet these requirements, the system will automatically perform the following transformations...

- Uppercase letters will be converted to lowercase
- Spaces will be converted to underscores `('_')`
- Special characters will be dropped

If the system can not automatically perform this transformation data will fail to load or load in a manner that is inconsistent with expectations.

## Dealing With File Layouts Changes

Succesful loading is dependent on each subsequent file delivery the following the established layout.

For example, `"*_transactions.csv"` data was following the following layout for most of the year:

```
    "ID","DATE","CUSTOMERID","PURCHASE","STOREID"
        "0122","December 10, 2015","123432100A","8.43","4985"
        "0123","December 10, 2015","543421111A","2.61","3212"
```

However, let us say that in November a change was made upstream which changed the layout. The `2016-11-15_transactions.csv` was delivered to `customers/transactions/` and the contents of the file contained a new entity called `LOYALTYID`:

```
    "ID","DATE","CUSTOMERID","PURCHASE","LOYALTYID","STOREID"
    "0122","December 10, 2015","123432100A","8.43","A102B","4985"
    "0123","December 10, 2015","543421111A","2.61","A904F","3212"
```

Due to the addition of `LOYALTYID` this creates a mismatch between the old structure and the new. This different layout would lead `2016-11-15_transactions.csv`, or any other file like it, to fail in loading to the `transactions` table because the underlying structure is different.

If this situation arises, please contact Openbridge support (support@openbridge.com).

## Directories

Log into your pipeline location and create a folder for each unique set of data you will be loading. As detailed in the previous examples, the name of the folder is very important as it will be the name of the table in the Redshift database (or other database) where you will access the data. It should therefore be a name that is descriptive enough for the end user to understand its contents. The folder name must also meet certain requirements. These requirements are outlined below...

- Must start with a letter
- Must contain only lowercase letters
- Can contain numbers and underscores `('_')` but no spaces or special characters (e.g. `'@'`, `'#'`, `'%'`)

Valid folder name:

```
    "customer_inquiries_jan_2014"
```

Invalid folder name:

```
     "Customer inquiries 1/14"
```

The Openbridge server will attempt to correct issues it detects with poor folder naming. For example, it will automatically remove spaces in names. A folder like this:

```
     "Customer crm file"
```

Would result in a folder called:

```
     "customercrmfile"
```

# Sending Compressed Files

Openbridge supports the delivery and processing of compressed files in `zip`, `gzip` or `tar.gz` formats. However, since we do not know the contents of an archive prior to it arriving and us unpackaging it, custom processing is needed to ensure we are handling the contents in the manner you want us to. This is especially important when a single archive contains a number of distinct files that have different data types and structures. Here are a few different use cases that arise with compressed files.

## Simple Use Case: One Archive, One File

In this example we have `*_rewards.zip` archives that are delivered to `customers/rewards/`.

The `2016-01-01_rewards.zip` archive contains three files; `2016-01-01a_rewards.csv`, `2016-01-01b_rewards.csv` and `2016-01-01c_rewards.csv`. The archive structure looks like this:

```
    customers/
      ├── rewards/
          ├── 2016-01-01_rewards.zip
              ├── 2016-01-01a_rewards.csv
              ├── 2016-01-01b_rewards.csv
              ├── 2016-01-01c_rewards.csv
```

The system will unpack `2016-01-01_rewards.zip` according to the contents of the archive. This would result in three directories, one for each file.

```
    customers/
      ├── rewards/
          ├── 2016-01-01_rewards.zip
          ├── 2016-01-01a_rewards/
              ├── 2016-01-01a_rewards.csv
          ├── 2016-01-01b_rewards/
              ├── 2016-01-01b_rewards.csv
          ├── 2016-01-01c_rewards/
              ├── 2016-01-01c_rewards.csv
```

The unpacked directory is based on the name of the file. For example, a filename of `2016-01-01a_rewards.csv` results in a directory named `2016-01-01a_rewards/`

In this example all `*_rewards.zip` archives share a common schema. This means `2016-01-01a_rewards.csv`, `2016-01-01b_rewards.csv` and `2016-01-01c_rewards.csv` share the same exact structure:

```
    "ID","DATE","CUSTOMERID","PURCHASE","LOYALTYID","STOREID"
      "0122","December 10, 2015","123432100A","8.43","A102B","4985"
      "0123","December 10, 2015","543421111A","2.61","A904F","3212"
```

So regardless of the unpacking directory hierarchy, everything in `rewards/` share a schema and will route all data to a common `rewards` table.

## Complex Use Case: One Archive, Many Files

In this example we have more complex archives called `*_offers.zip`. These archives are delivered to `customers/offers/ofers_targeted`.

The `2016-01-01_offers.zip` archive contains three **different** files; `2016-01-01_source.csv`, `2016-01-01_campaigns.csv` and `2016-01-01_costs.csv`.

The unpacked archive structure would follow the same pattern as the simple file example. Each file would be unpacked into its own directory with the name of the directory use the name of the file.

```
    customers/
      ├── offers/
          ├── offers_open/
          └── offers_targeted/
              ├── 2016-01-01_offers.zip
                  ├── 2016-01-01_source/
                      ├── 2016-01-01_source.csv
                  ├── 2016-01-01_campaigns/
                      ├── 2016-01-01_campaigns.csv
                  ├── 2016-01-01_costs/
                      ├── 2016-01-01_costs.csv
```

The files `2016-01-01_source.csv`, `2016-01-01_campaigns.csv` and `2016-01-01_costs.csv` have the following structure.

Source

```
    "ID","DATE","CHANNELID", "CAMPAIGNID","PARTNERID"
    "0122","December 10, 2015","123432100A","8.43","A102B","4985"
    "0123","December 10, 2015","543421111A","2.61","A904F","3212"
```

Campaigns

```
    "ID","DATE","CAMPAIGNID","RESPONSE","ROI","OFFERID"
    "0122","December 10, 2015","123432100A","8.43","A102B","4985"
    "0123","December 10, 2015","543421111A","2.61","A904F","3212"
```

Costs

```
    "ID","DATE","SPEND","BUDGET","ADID","CAMPAIGNID"
    "0122","December 10, 2015","123432100A","8.43","A102B","4985"
    "0123","December 10, 2015","543421111A","2.61","A904F","3212"
```

Unlike the earlier `rewards` example, these files represent three related, but different classes of data. This means they are not stored into a single offers table like the `rewards` example due to these differences.

The table names are user defined. This allows flexibility in how you want to handle this situation. For example, you may want `2016-01-01_source.csv`, `2016-01-01_campaigns.csv` and `2016-01-01_costs.csv` routed to `offers_source`, `offers_campaigns` and `offers_costs` tables.

To properly support delivery of complex compressed files, please contact Openbridge support so the proper handling and routing of compressed data can occur.

# Encoding Your Files

Openbridge supports UTF-8 filename character sets. It also supports ASCII, a subset of UTF-8\. The system will perform any character set decoding checks as the file is being delivered and will automatically discover the encoding used.

While we may accept the delivery of a file, there is no guarantee that downstream data processing pipelines will work when sending data that is _not_ UTF-8 encoded. A data pipeline will automatically attempt to convert encoding to UTF-8\. However, it might not work in all cases. We suggest, if feasible, to send data UTF-8 pre-encoded. Most systems generate UTF-8 encoded data.

## Check Encoding

Not sure how to check encoding? Here is an exaple using the command line. To check the encoding of a file called `foo.csv`, in your terminal you can type the following;

For Linux you would type:

```
    $ file -i foo.csv
```

For OS X it is almost the same command:

```
    $ file -I foo.csv
```

The output should look contain a `charset=` which indicates the character encoding for the file. In the case of our `foo.csv` the results are as follows:

```

    $ text/plain; charset=us-ascii
```

Notice the `charset=us-ascii` says the file is ASCII. Perfect! The file is ready to be transferred.

# How To Deliver Data

## Transfer Protocols

Pipeline supports three transfer methods: `SFTP`, `FTPES` and `FTP`. These are simple methods for transferring files between systems. In almost all cases, `SFTP` (or `FTPES`) is preferable to `FTP` because of its underlying security features. `FTP` is an insecure protocol that should only be used in limited cases or on networks you trust. `SFTP` , `FTPES` and `FTP` is integrated into many graphical tools or can be done directly via the command line

### Secure File Transfer Protocol (SFTP)

`SFTP` is based on the SSH2 protocol, which encodes activity over a secure channel. Unlike FTP, SSH2 only uses a single TCP connection and sends multiple transfers, or "channels", over that single connection.

Openbridge currently supports public key and password authentication for both `SFTP` and SCP file transfer protocols. If you need to use public key authentication plaese submit a support ticket and we can set that up for you. We do NOT support any shell access. Also, not all `SFTP` commands are accepted.

Connection Details:

```
    Host:               pipeline.openbridge.io
    Port:               2222
    Protocol:           SFTP
    User:               Provided separately
    Password:           Provided separately
```

### FTP Explicit Mode (TLS)/(SSL)

Openbridge supports FTP explicit mode (also known as `FTPES`). Your `FTPES` client must be configured to "explicitly request" SS/TLS security.

The system includes support for most TLS and SSL cryptographic protocols. The system uses OpenSSL cryptographic "engine", a module within the OpenSSL library. OpenSSL will find from the supported engines the first one usable for the connection. If no usable engines are found, OpenSSL will default to its normal software implementation.

SSL/TLS protocol versions used when establishing SSL/TLS sessions is **TLSv1, TLSv1.1 and TLSv1.2**. Please note SSLv3 and SSv2 are not permitted due to security gaps. You can read more about this [here][1] and [here][2]

If your client connects to the FTPS server with an unknown security mechanism, the server will respond to the AUTH command with error code 504 (not supported).

Connection Details:

```
    Host:             pipeline.openbridge.io
    Port:             21
    Protocol:         FTPES
    User:             Provided separately
    Password:         Provided separately
```

### File Transfer Protocol (FTP)

Openbridge does support the use of `FTP`. We recognize that there are some systems that can only deliver data via `FTP`. For example, many of the Adobe export processes typically occur via `FTP`. However, it should be noted that the use of `FTP` offers **no encryption** for connection and data transport. Using the `FTP` protocol is regarded to be unsafe. It is therefore advisable to use `SFTP` or `FTPES` connections to ensure that data is securely transferred.

Connection Details:

```
    Host:             pipeline.openbridge.io
    Port:             21
    Protocol:         Plain FTP
    User:             Provided separately
    Password:         Provided separately
```

## Check Your Firewall

If you are having connection difficulties, please make sure you can make outbound network connections for `FTPES` and `FTP` via `port:21`. For those using `SFTP` make sure outbound `port:2222` is open.

## Blocked Files

To help protect the integrity of the data sent to Openbridge, we do not allow delivery of certain files. For example, you are now allowed to sedn a file called `app.exe` or `my.sql`. This reduces the potential for introducing a unwanted or malicious software threats. The following is a sample of the blocked files:

```
    ade|adp|app|ai|asa|ashx|asmx|asp|bas|bat|cdx|cer|cgi|chm|class|cmd|com|config|cpl|crt|csh|dmg|doc|docx|dll|eps|exe|fxp|ftaccess|hlp|hta|htr|htaccess|htw|html|htm|ida|idc|idq|ins|isp|its|jse|ksh|lnk|mad|maf|mag|mam|maq|mar|mas|mat|mau|mav|maw|mda|mdb|mde|mdt|mdw|mdz|msc|msh|msh1|msh1xml|msh2|msh2xml|mshxml|msi|msp|mst|ops|pdf|php|php3|php4|php5|pcd|pif|prf|prg|printer|pst|psd|rar|reg|rem|scf|scr|sct|shb|shs|shtm|shtml|soap|stm|tgz|taz|url|vb|vbe|vbs|ws|wsc|wsf|wsh|xls|xlsx|xvd
```

If you attempt to send a file that matches a blocked file then the transfer will not be allowed.

## Hidden Files

Hidden files are not allowed on the server. Hidden files have a dot prefix `.` in the file name. For example, `.file.txt` or `.foo` would be considered hidden files and be blocked.

The following are examples of names using a dot prefix:

```
    .file.txt
    .file.csv
    .file
    ..file
```

If you attempt to send a file that contains a `.` prefix the transfer will not be allowed.

# Error Handling

In most cases, things work without incident. However, there are situations where an error may arise. Troubleshooting an error can be a difficult and time consuming endeavor, especially in cases where a system may be sending 1000s of files a day.

## Bulk Transfers

Openbridge does not have visibility into the what _should_ be sent from a source system. It only knows what _was_ sent by the source system. For example, if there are 100 files in a source system and only 50 were sent, Openbridge is only aware of the 50 delivered files. We will not know that an additional 50 files were not delivered.

### Manifests

The source system should be tracking what was delivered and what was not delivered. We suggest that a manifest of files is maintained in the source system. This manifest would indeityf the files to be delivered and their state (success? failure? pending?). The manifest procedure allows the source system to recover from certain errors such as failure of network, source/host system or in the transfer process. In the event of an error, the source system would know to attempt a redeliver for any file that had not received a successful "226" code from Openbridge.

## Status Codes

Your client will normally be sent a response code of "226" to indicate a successful file transfer. However, there are other status codes that may be present. See below:

- A code 226 is sent if the entire file was successfully received and stored
- A code 331 is sent due to a login error. The name okay, but you are missing a password.
- A code 425 is sent if no TCP connection was established. This might mean the server is not available of some other network error. Change from PASV to PORT mode, check your firewall settings
- A code 426 is sent if the TCP connection was established but then broken by the client or by network failure. Retry later.
- A request with code 451, 452, or 552 if the server had trouble saving the file to disk. Retry later
- A 553 code means the requested action not taken because the file name sent is not allowed. Change the file name or delete spaces/special characters in the file name.

If you are stuck with an error condition, reach out to support for help.

## File Integrity

Openbridge uses checksum (or hashes) validation to ensure the integrity of the file transfer. Openbridge a 128-bit MD5 checksum, which is represented by a 32-character hexadecimal number. This only applies to files once they are in our possession. We suggest that source systems also calculate MD5 checksum in advance of file delivery to increase the confidence that the integrity of the file to be delivered is met. Depending the tools employed by a source system, the checksum process might be built-in.

Employing file integrity checks will help you cross check that the files delivered form the source system match what was delivered to Openbridge.

While the use of MD5 is the default, Openbridge can support other checksum commands when sending data to us:

- XCRC (requests CRC32 digest/checksum)
- XSHA/XSHA1 (requests SHA1 digest/checksum)
- XSHA256 (requests SHA256 digest/checksum)
- XSHA512 (requests SHA512 digest/checksum)

If you need support for any of these these, please contact support.

# Connection Considerations

## DNS-based Blackhole List (DNSBL) / Real-time Blackhole List (RBL)

Openbridge employs a DNSBL (commonly known as a 'Blocklist"). This is a database that is queried in realtime for the purpose of obtaining an opinion on the origin of incoming hosts. The role of a DNSBL is to assess whether a particular IP Address meets acceptance policies of inbound connections. DNSBL is often used by email, web and other network services for determining and rejecting addresses known to be sources of spam, phishing and other unwanted behavior.

More information on DNS blacklists can be found here:(<http://en.wikipedia.org/wiki/DNSBL>)

## Anti-Virus, Malware and Trojans

Openbridge employs an anti-virus toolkit to scan for viruses, trojans, and other questionable items to prevent them from being uploaded to our system. The process is designed to detect in real-time threats present in any incoming files. This means any file sent to Openbridge will be scanned as it is streamed to us prior to allowing the file upload to be fully written to the filesystem.

Any files uploaded meeting the criteria as a threat will result the transfer being rejected.

## Account Ban and Lockout

Openbridge employs a dynamic "ban" lists that prevents the banned user or host from logging in to the server. This will occur if our system detects 4 incorrect login attempts. The ban will last for approximately 30 minutes at which time you can attempt to login again. If you continue to have difficulties please contact support.

## Idle Connection Time Limits

Openbridge sets the maximum number of seconds a connection between the our server and your client can exist after the client has successfully authenticated. This is typically 10 minutes or 600 seconds in length. If you are idle for longer than this time allotment the server will think you are finished and disconnect your client. If this occurs you will simply need to reconnect.

# Reference

## FTP Clients

There are plenty of FTP clients to choose from. They are super easy to use and provide a wealth of features to help you manage the process of transferring files. There are also CLI tools or programatic methods of files transfer too.

## GUI

### Free

- [WinSCP](https://winscp.net/), aka Windows Secure Copy, is a free, open-source FTP client.
- [FireFTP](http://fireftp.net/) is a Firefox extension that integrates a powerful FTP client directly into our favorite browser.
- [Cyberduck](https://cyberduck.io) is a free, open-source FTP client for Mac OS X
- [FileZilla](https://filezilla-project.org/) is a free, open-source FTP client for Windows, Mac, and Linux.

### Paid

- [Transmit](https://panic.com/transmit/) Mac OS X client packed to the brim with innovative features.
- [SmartFTP](https://www.smartftp.com) is Windows FTP (File Transfer Protocol), FTPS, SFTP, WebDAV, S3, Google Drive, OneDrive, SSH, Terminal client.
- [CuteFTP](https://www.globalscape.com/cuteftp) sits alongside FileZilla as the best-known names in FTP. Cost is about $40.

## CLI

If you prefer the command line, then is curl (<https://curl.haxx.se/>) is an excellent choice. curl is used in command lines or scripts to transfer data. Here are some sample code snippets showing a bash function leveraging curl:

Sending a file via FTP

```bash
function ftp_send_file() {
    echo "STARTING: FTP testing..."
    ( curl -T /home/customers.csv ftp://pipeline.openbridge.io:21/customer/ -u user:password )
    if [ $? -ne 0 ]; then echo "ERROR: FTP transfer test failed" && exit 1; else echo "success"; fi
}
```

Send file via SFTP

```bash
function sftp_send_file() {
    echo "STARTING: SFTP testing...."
    ( curl -T /home/customers.csv -u user:password sftp://pipeline.openbridge.io:2222/customer/ -k )
    if [ $? -ne 0 ]; then echo "ERROR: SFTP transfer test failed" && exit 1; else echo "success"; fi
}
```

## Python

You can embed file transfers into your programs too. FOr example, Paramiko is a great choice for Python users. What is [Paramiko](http://www.paramiko.org/)? Paramiko is a Python (2.6+, 3.3+) implementation of the SSHv2 protocol, providing both client and server functionality. While it leverages a Python C extension for low level cryptography (Cryptography), Paramiko itself is a pure Python interface around SSH networking concepts.

Here is a [demo](https://github.com/paramiko/paramiko/blob/master/demos/demo_sftp.py) of a SFTP client written with Paramiko.

# FAQs

## Why isn't the data I posted to the pipeline loaded to Redshift?

If you don’t see data that you are expecting to see in one or more Redshift tables, there are a few steps you can take to diagnose the issue…

1. Verify that the file(s) containing the data in question was successfully posted to the FTP location.  Most FTP software includes search functionality that allows  you to search the relevant folder for the filename or a portion of the filename  (e.g. filename contains ‘foo’)

   **Resolution:** If the file was not posted to the FTP location, attempt to re-post (or re-request) the data and check the Redshift table to see if it has loaded.  Depending on the size of the file and other files in the processing queue, this could take anywhere from a few seconds to a several minutes.

2. If the file was successfully posted to the FTP location, download a copy of the file to your desktop and open the file in a text editor (e.g. Notepad) or Excel to check the following issues that could result in a load failure:

  1. Does the file contain data?
  2. Is the file layout (i.e. columns and column order) the same as its corresponding Redshift table?
  3. Is the file delimited properly (e.g. tab or comma-quote) and are the delimiters consistent with the initial load file for the table?
  4. Are the values for each field formatted properly (e.g. text, number, date) and consistent with the initial load file for the table?

   **Resolution:** Fix the error(s) in the file, re-post the file to the FTP location with a new name (e.g. if original file was named 'some_data_20150101.csv' rename new file to something like 'some_data_20150101_2.csv') and check the Redshift table to see if it has been successfully loaded.

3. If the file passes the above checks, please submit a support ticket by emailing support@openbridge.com so that a support agent can assist with the diagnosis and resolution of the load error.  To facilitate the diagnostic efforts please be sure to include the following in the email:

  1. Redshift table name with the missing data
  2. Criteria that can be used to identify the missing row(s) in the table (e.g. table ‘foo’ is missing data for the date = xx/xx/xxxx)
  3. Filename (and FTP path name if known) for the unloaded data

The initial load file is the first file that is posted to a particular folder.  This file is what generates and defines the properties for the respective Redshift table (field names, order and formats).  Subsequent files posted to a folder need to have the same specifications as the initial load file to be loaded successfully. 