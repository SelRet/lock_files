lock_files
==========
[![Releases](https://img.shields.io/github/release/jlinoff/lock_files.svg?style=flat)](https://github.com/jlinoff/lock_file/releases)

This is a python command line tool to lock (encrypt) or unlock (decrypt) files using AES
encryption and a common password. This version works in python2 and python3.

## Overview
You can use it to lock files before they are uploaded to storage services like DropBox or Google Drive.

The files are locked/encrypted using AES-CBC mode from the pycrypto package (`import Crypto`) so the source is useful for understanding
how to use this package. The encrypted data is encoded in base64 and broken into 72 character lines so that the output is ASCII with
no very long lines. That makes it convenient for copying. You can change the line length using the `-w` option.

To encrypt files you need to specify a password. The password can be stored in a safe file (`-p`), specified on the command line in plaintext (`-P`) or it can be manually entered each time the tool is run from a password prompt.

In the examples below, `-P` is used to specify a password on the command line in plaintext. This is only for convenience. Normally
specifying a plaintext password is a bad idea for any production work because it will show up in the shell history.

The tool checks each file to make sure that it is writeable before processing. If any files are not writeable,
it means that they cannot be changed so the program aborts unless you specified the continue `-c` option. Files up until that point
are locked but they can easily be unlocked.

You can use the `-v -v` option to see details about each file being processed.

You can specify `-j` to increase or decrease the number of threads. This program, like all Python programs, is subject to the
limitations of the Global Interpreter Lock (GIL) so your multi-threading performance improvement may not be what you
expect and may be different between Python 2.7 and 3.x.

You can specify `-c` to generate files that are compatible with `openssl`.

The program is re-entrant which means that you can run lock a single file multiple times with different passwords. 
Each new password will append an additional `.locked` extension. If you don't like the `.locked` extension, you can change it
using the `-s` (suffix) option.

### Simple Lock/Unlock
Lets start with the simplest case, locking and unlocking a simple file using the password `secret`. The file is `file.txt`.

```bash
$ ls file.txt*
file.txt
$ cat -n file.txt
     1	Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do
     2	eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad
     3	minim veniam, quis nostrud exercitation ullamco laboris nisi ut
     4	aliquip ex ea commodo consequat. Duis aute irure dolor in
     5	reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla
     6	pariatur. Excepteur sint occaecat cupidatat non proident, sunt in
     7	culpa qui officia deserunt mollit anim id est laborum.
$ lock_files.py -P secret --lock file.txt
$ ls file.txt*
file.txt.locked
$ cat -n file.txt.locked
     1	mkpk82pjztGcqTD5y1MnM62Lb8M8+ccS5lB8+YHSqoB3BwrZ+ahzyEjqoEbqMNHvlFqZPYAh
     2	CQzhtnDodCca1tt/sJ6vaHIUp10LNVQmxUN2N5tbAZ8YAvihbG6c/t0x3qbUvwLy/HI4+Dig
     3	RAoASHSSNuTQJTdDNqvx4mD7bs/GGj1ljBYWPcWR04s780gY3JO8BIL8B0jvHgjwofc596MM
     4	3ItylXw97/On6iqCJzToyxNPPGc11tkEiR98YON2kArk8JWnKhi7RZOoXJCD9/dS0EeIMOp1
     5	Bi71EXmBAgWsHWtfs9NUCyQViCHUX9WMqJWdsIWP0LVhsqcfyUUxuruZD6VpcEc2A0puTNGF
     6	Jh2n9jUqdcLDv+Y/FfD2eYzoSWp+vHyNpP2qePD7jRepWfIYqsvMyqIhfktIi/dyeM28X+Nx
     7	v4+2QWtjz9PSOvzUwFG28NyUONeaJzwLZa8A8HFQPl/zIraqpZg34iRZJkHQtPZ3aveI9yc+
     8	E+6esv/Wo07qiffozyEykrOT+fJ1HbhTdkzjZtq1fQxP9LE367zjwdzex99dTL14dHcSN9kW
     9	2s4Qxg4ZQ8eAqwLX3c4VnIqUeh0aqqjKJqKqftjDBVg=
$ lock_files.py -P secret --unlock file.txt.locked
$ ls file.txt*
file.txt
```

From this example you can see that the input file is encrypted/locked and is stored in `file.txt.locked`. It is then decrypted/unlocked
back to the original file `file.txt`. This is the normal mode of operation. 

There is another mode called _in place_ that does away with the `.locked` extgension. It is explained and demonstrated a bit later
but before talking about that in detail, it is important to note that the above example can be done just as easily using a common
tool like `openssl` as follows.

```bash
$ openssl aes-256-cbc -pass pass:secret -e -a -salt -in file.txt -out file.txt.lock
$ openssl aes-256-cbc -pass pass:secret -d -a -salt -in file.txt.lock -out file.txt
```

So why use lock_files.py?

For a single file _there is probably no good reason_ but because it handles multiple files and directories,
you definitely want to consider using it for groups of files. 

> I suppose that an argument could be made for a single file because
> the command line is a bit simpler but it is definitely not a strong argument.

Here is an example that shows how to lock groups of files. It locks all of the files in the secrets directory and all files with
the `.confidential` extension:

```bash
$ lock_files.py -P secret -v -v --lock secrets *.confidential
```

And here is how you unlock them.

```bash
$ lock_files.py -P secret -v -v --unlock secrets *.confidential.locked
```

Note that for directories, you don't need to worry about the `.locked` extension.

### In Place Mode
Now lets consider the _in place_ mode. The term _in place_ means not appending the `.locked` suffix to each file name that was locked.
Here is how you would use this tool to lock/unlock files _in place_ using the above example with
the `secret` directory and the files with the `.confidential` extensions.

```bash
$ lock_files.py -P secret -i --lock secrets *.confidential
$ lock_files.py -P secret -i --unlock secrets *.confidential
```

Note that you do _not_ have to use the `.locked` extension here because _it doesn't exist_. Each locked file has the same name as the
unlocked file.

> Note that _in place_ is not secure because data will be lost if the disk fills up during a write operation
> and it is not able to complete.

Here is how you could use _in place_ mode to decrypt a file, execute a program and then re-encrypt it when the program exits.

    $ lock_files.py -p ./password -i -u file1.txt
    $ edit file1.txt
    $ lock_files.py -p ./password -i -l file1.txt

This approach can be used to make sure that source files are always locked/encrypted when not in use.

### Password Files
Here is how you could generate a password file and use it to lock and unlock files.

```bash
$ cat -n file.txt
     1	Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do
     2	eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad
     3	minim veniam, quis nostrud exercitation ullamco laboris nisi ut
     4	aliquip ex ea commodo consequat. Duis aute irure dolor in
     5	reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla
     6	pariatur. Excepteur sint occaecat cupidatat non proident, sunt in
     7	culpa qui officia deserunt mollit anim id est laborum.
$ LC_CTYPE=C tr -dc A-Za-z0-9_\- < /dev/urandom | head -c 32  | args >passfile
$ chmod 0600 passfile
$ cat passfile
uWE8j_iwFabRrzZQu_0_sWecH1zJZf7p
$ lock_files.py -p passfile -l file.txt
$ cat file.txt.locked 
     1	8b/KZ+3TEEdEVm51FPLcj6CDdAoN3QlFQ208IPBa3qb58zWQ+X9K/ZKHPKPweA1pemH83XSA
     2	FYGaU8LMx5wCcf7yzhf/hMiFOR5MNoAONsZ58e9BJRz4Q35oYiyA4z2o5TK485NEekKqAU8j
     3	0UiYjOfH1zkX56LPR3cas+rnxQQA1srXtnvY7XwaGJx856sQQ3hgpfHnjTkvMo3wfiwDv0Lm
     4	gRCQigDuZaa222g+fZW7hGRluwtrS7/6QX40J78/jLNehJnioc+tRLJyHJQUAO3QkA+DVODS
     5	BCtaCa+ueVRpcfcuOKCvNLuwV82RQSR3KW+EZWq3hphSqNRp7iYZne0hD5uRJHs1uu2tm6ca
     6	HmwgGpy+yfcuYYRBmwQy9kMRzNB6xN8IEH4Mo0blCfVReKNQELE9gOfL1g3LMM62/a8IROhw
     7	g3h/Rsh//vlstE/3FlSQnzhZ9o6CItz/TjCRQ9oiBvQeVrZTZ1tkoiWeysY/DGnfc+Y+rSMG
     8	m9GowOdzfPvLWzxJbEN8pj3LRYplowppZykySXQl7L6WVqGMg+wu1qhLyzMN1E6EUXyQNrXN
     9	JAV1Pp+Ix+t9QQKkjh3+fEHkY++Ki69GxKnARFWrTtk=
$ lock_files.py -p passfile -u file.txt.locked
$ cat -n file.txt
     1	Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do
     2	eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad
     3	minim veniam, quis nostrud exercitation ullamco laboris nisi ut
     4	aliquip ex ea commodo consequat. Duis aute irure dolor in
     5	reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla
     6	pariatur. Excepteur sint occaecat cupidatat non proident, sunt in
     7	culpa qui officia deserunt mollit anim id est laborum.
```
### Openssl compatibility
By default the files encrypted by lock_files.py are not compatible with openssl. However, if you want your encrypted files to
be decrypted by `openssl` or if you want lock_files.py to be able to unlock files that were decrypted by `openssl`, 
you can use compatibility mode (`-c`).

The example below shows how to use lock_files.py encrypt a file in compatibility mode and then decrypt it using openssl.

```bash
$ lock_files.py -c -P secret -l file.txt
$ openssl enc -aes-256-cbc -d -a -pass pass:secret -in file.txt.locked -out file.txt
```

The example below shows how to use openssl to encrypt a file and then decrypt it using lock_files.py.

```bash
$ openssl enc -aes-256-cbc -e -a -pass pass:secret -in file.txt -out file.txt.locked
$ lock_files.py -c -P secret -u file.txt.locked
```

When `-c` is specified on the command line, all files encrypted or decrypted will be able to be processed by openssl.

## Download and Test
Here is how you download and test it. I have multiple versions of python installed so I set the the first argument
to the test script. If you only have a single version of python, the you do not specify an argument. It assumes the 
python that is in your path.

```bash
$ git clone https://github.com/jlinoff/lock_files.git
$ cd lock_files/test

$ # Use the default version of python.
$ ./test.sh

$ # Use a specific version of python 2.
$ ./test.sh 'python2.7 ./test.sh'
[output snipped]

$ # Use a specific version of python 3.
$ ./test.sh 'python3.6 ./test.sh'
[output snipped]
```

## Help
Here is the on-line help. It describes all of the options and provides examples.

```bash
$ lock_files.py -h
USAGE:
  lock_files.py [OPTIONS] [<FILES_OR_DIRS>]+

DESCRIPTION:
  Encrypt and decrypt files using AES encryption and a common
  password. You can use it lock files before they are uploaded to
  storage services like DropBox or Google Drive.
  
  The password can be stored in a safe file, specified on the command
  line or it can be manually entered each time the tool is run.
  
  Here is how you would use this tool to encrypt a number of files using
  a local, secure file. You can optionally specify the --lock switch but
  since it is the default, it is not necessary.
  
     $ lock_files.py file1.txt file2.txt dir1 dir2
     Password: secret
     Re-enter password: secret
  
  When the lock command is finished all of files will be locked (encrypted,
  with a ".locked" extension).
  
  You can lock the same files multiple times with different
  passwords. Each time lock_files.py is run in lock mode, another
  ".locked" extension is appended. Each time it is run in unlock mode, a
  ".locked" extension is removed. Unlock mode is enabled by specifying
  the --unlock option.
  
  Of course, entering the password manually each time can be a challenge.
  It is normally easier to create a read-only file that can be re-used.
  Here is how you would do that.
  
     $ cat >password-file
     thisismysecretpassword
     EOF
     $ chmod 0600 password-file
  
  You can now use the password file like this to lock and unlock a file.
  
     $ lock_files.py -p password-file file1.txt
     $ lock_files.py -p password-file --unlock file1.txt.locked
  
  In decrypt mode the tool walks through the specified files and
  directories looking for files with the .locked extension and unlocks
  (decrypts) them.
  
  Here is how you would use this tool to decrypt a file, execute a
  program and then re-encrypt it when the program exits.
  
     $ # the unlock operation removes the .locked extension
     $ lock_files.py -p ./password --unlock file1.txt.locked
     $ edit file1.txt
     $ lock_files.py -p ./password file1.txt
  
  The tool checks each file to make sure that it is writeable before
  processing. If any files is not writeable, the program reports an
  error and exits unless you specify --warn in which case it
  reports a warning that the file will be ignored and continues.
  
  If you want to change a file in place you can use --inplace mode.
  See the documentation for that option to get more information.
  
  If you want to encrypt and decrypt files so that they can be
  processed using openssl, you must use compatibility mode (-c).
  
  Here is how you could encrypt a file using lock_files.py and
  decrypt it using openssl.
  
     $ lock_files.py -P secret --lock file1.txt
     $ ls file1*
     file1.txt.locked
     $ openssl enc -aes-256-cbc -d -a -pass pass:secret -in file1.txt.locked -out file1.txt
  
  Here is how you could encrypt a file using openssl and then
  decrypt it using lock_files.py.
  
     $ openssl enc -aes-256-cbc -e -a -pass pass:secret -in file1.txt -out file1.txt.locked
     $ ls file1*
     file1.txt      file1.txt.locked
     $ lock_files.py -c -W -P secret --unlock file1.txt.locked
     $ ls file1*
     file1.txt
  
  Note that you have to use the -W option to change errors to
  warning because the file1.txt output file already exists.

POSITIONAL ARGUMENTS:
  FILES                 files to process

OPTIONAL ARGUMENTS:
  -h, --help            Show this help message and exit.
                         
  -c, --openssl         Enable openssl compatibility.
                        This will encrypt and decrypt in a manner
                        that is completely compatible openssl.
                        
                        This option must be specified for both
                        encrypt and decrypt operations.
                        
                        These two decrypt commands are equivalent.
                           $ openssl enc -aes-256-cbc -d -a -pass pass:PASSWORD -in FILE -o FILE.locked
                           $ lock_files.py -P PASSWORD -l FILE
                        
                        These two decrypt commands are equivalent.
                           $ openssl enc -aes-256-cbc -e -a -pass pass:PASSWORD -in FILE.locked -o FILE
                           $ lock_files.py -P PASSWORD -u FILE
                         
  -d, --decrypt         Unlock/decrypt files.
                        This option is deprecated.
                        It is the same as --unlock.
                         
  -e, --encrypt         Lock/encrypt files.
                        This option is deprecated.
                        This is the same as --lock and is the default.
                         
  -i, --inplace         In place mode.
                        Overwrite files in place.
                        
                        It is the same as specifying:
                           -o -s ''
                        
                        This is a dangerous because a disk full
                        operation can cause data to be lost when a
                        write fails. This allows you to duplicate the
                        behavior of the previous version.
                         
  -j NUM_THREADS, --jobs NUM_THREADS
                        Specify the maximum number of active threads.
                        
                        This can be helpful if there a lot of large
                        files to process where large refers to files
                        larger than a MB.
                        
                        Default: 8
                         
  -l, --lock            Lock files.
                        Files are locked and the ".locked" extension
                        is appended unless the --suffix option is
                        specified.
                         
  -n, --no-recurse      Do not automatically recurse into
                        subdirectories.
                         
  -o, --overwrite       Overwrite files that already exist.
                        This can be used in conjunction disable file
                        existence checks.
                        
                        It is used by the --inplace mode.
                         
  -p PASSWORD_FILE, --password-file PASSWORD_FILE
                        file that contains the password.
                        The default behavior is to prompt for the
                        password.
                         
  -P PASSWORD, --password PASSWORD
                        Specify the password on the command line.
                        This is not secure because it is visible in
                        the command history.
                         
  -s EXTENSION, --suffix EXTENSION
                        Specify the extension used for locked files.
                        Default: .locked
                         
  -u, --unlock          Unlock files.
                        Files with the ".locked" extension are
                        unlocked.
                        
                        If the --suffix option is specified, that
                        extension is used instead of ".locked".
                         
  -v, --verbose         Increase the level of verbosity.
                        A single -v generates a summary report.
                        
                        Two or more -v options show all of the files
                        being processed.
                         
  -V, --version         Show program's version number and exit.
                         
  -w INTEGER, --wll INTEGER
                        The width of each locked/encrypted line.
                        This is important because text files with
                        very, very long can sometimes cause problems
                        during uploads. If set to zero, no new lines
                        are inserted.
                        
                        Default: 72
  -W, --warn            Warn if a single file lock/unlock fails.
                        Normally if the program tries to modify a
                        fail and that modification fails, an error is
                        reported and the programs stops. This option
                        causes that event to be treated as a warning
                        so the program continues.
                         

EXAMPLES:
   # Example 1: help
   $ lock_files.py -h

   # Example 2: lock/unlock a single file
   $ lock_files.py -P 'secret' file.txt
   $ ls file.txt*
   file.txt.locked
   $ lock_files.py -P 'secret' --unlock file.txt
   $ ls -1 file.txt*
   file.txt

   # Example 3: lock/unlock a set of directories
   $ lock_files.py -P 'secret' project1 project2
   $ find project1 project2 --type f -name '*.locked'
   <output snipped>
   $ lock_files.py -P 'secret' --unlock project1 project2

   # Example 4: lock/unlock using a custom extension
   $ lock_files.py -P 'secret' -s .EncRypt file.txt
   $ ls file.txt*
   file.txt.EncRypt
   $ lock_files.py -P 'secret' -s .EncRypt --unlock file.txt

   # Example 5: lock/unlock a file in place (using the same name)
   #            The file name does not change but the content.
   #            It is compatible with the default mode of operation in
   #            previous releases.
   #            This mode of operation is not recommended because data
   #            will be lost if the disk fills up during a write.
   $ lock_files.py -P 'secret' -i -l file.txt
   $ ls file.txt*
   file.txt
   $ lock_files.py -P 'secret' -i -u file.txt
   $ ls file.txt*
   file.txt

   # Example 6: use a password file.
   $ echo 'secret' >pass.txt
   $ chmod 0600 pass.txt
   $ lock_files.py -p pass.txt -l file.txt
   $ lock_files.py -p pass.txt -u file.txt.locked

   # Example 7: encrypt and decrypt in an openssl compatible manner
   #            by specifying the compatibility (-c) option.
   $ echo 'secret' >pass.txt
   $ chmod 0600 pass.txt
   $ lock_files.py -p pass.txt -c -l file.txt
   $ # Dump the locked password file contents, then decrypt it.
   $ openssl enc -aes-256-cbc -d -a -pass file:pass.txt -in file.txt.locked
   $ lock_files.py -p pass.txt -c -u file.txt.locked

COPYRIGHT:
   Copyright (c) 2015 Joe Linoff, all rights reserved

LICENSE:
   MIT Open Source

PROJECT:
   https://github.com/jlinoff/lock_files
```
