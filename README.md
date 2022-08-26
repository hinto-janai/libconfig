# libconfig
>Functions for handling config files in Bash

## Contents
- [About](#About)
- [Speed](#Speed)
- [Input](#Input)
	- [config::source](#ConfigSource)
	- [config::carry](#ConfigCarry)
- [Example](#Example)
	- [config::source](#ConfigSource-1)
	- [config::carry](#ConfigCarry-1)
- [Errors](#Errors)
	- [config::source](#ConfigSource-2)
	- [config::carry](#ConfigCarry-2)

## About
These functions handle common config file operations in Bash.

| function           | purpose | dependency |
|--------------------|---------|------------|
| `config::source()` | Parses and sources a text file without executing the code from the file itself, like the unsafe `source` does | `Bash v4.4+` |
| `config::carry()`  | Carrys old variable values from one file to another and prints merged config file to STDOUT | `sed`, `grep` |

## Speed
`config::source()` scales around ***2-40x~ more poorly*** than `source` as the number of lines in the config file increase, and as the arguments you're looking for increases.

The nominal time difference however, is small. Using `config::source()` vs `source` on 100 lines will be an un-noticable difference.

| lines of text | source time | config::source time | config::carry() time |
|---------------|-------------|---------------------|----------------------|
| 10            | 0.000s      | 0.001s              | 0.006s
| 100           | 0.001s      | 0.010s              | 0.020s
| 1000          | 0.002s      | 0.080s              | 0.200s

## Input
### `config::source()`
Needs at the minimum 3 arguments:
- `CONFIG_PATH`
- `DATA_TYPE`
- `OPTION_NAME`

With every 2 pair argument after that being another option you'd like to read from the same config file.

| Data Type | Valid Input                                                  | [[ Bash =~ Regex Pattern ]]      |
|-----------|--------------------------------------------------------------|----------------------------------|
| ip        | integers seperated by '.'; must begin/end with integer       | ^${2}=[0-9.]+'.'[0-9]+$          |
| int       | any positive integer including 0; no floating point          | ^${2}=[0-9]+$                    |
| port      | like ip, but expects a port ':'; must begin/end with integer | ^${2}=[0-9:.]+'.'[0-9]':'[0-9]+$ |
| bool      | true/false                                                   | ^${2}=true$ OR ^${2}=false$      |
| char      | alphanumeric and some extra characters                       | ^${2}=[[:alnum:]._-]+$           |
| path      | alphanumeric seperated by '/'; no spaces allowed             | ^${2}=[[:alnum:]./_-]+$          |

Here's a typical configuration file you'd like to read:
```bash
MY_IP=127.0.0.1
MY_INT=4
MY_PORT=127.0.0.1:80
MY_BOOL=true
MY_CHAR=hello
MY_PATH=/home/hinto
```

To parse this with `config::source()`, it would look like:
```bash
config::source /path/to/config \
	ip   MY_IP   \
	int  MY_INT  \
	port MY_PORT \
	bool MY_BOOL \
	char MY_CHAR \
	path MY_PATH
```
`config::source()` will read the file provided line-by-line and look for `OPTION_NAME=value`. If the value is empty, or does not match the data type provided (MY_IP=not_an_ip), then it will not intialize that option as a variable.

If it matches (MY_BOOL=true), it will intialize it as a global variable: `declare -g MY_BOOL=true`

If multiple arguments are given in the config file, for example:
```bash
MY_BOOL=true
MY_BOOL=false
```
The latest version will be used, `MY_BOOL=false`

All quote characters `"` and `'` will be stripped before parsing, so a config file can use them or not.  
Empty spaces will also be stripped:
```bash
MY_BOOL="true"
MY_BOOL='true '
MY_BOOL=true
```
These all work.

---

### `config::carry()`
Needs exactly 2 arguments, an old file and a new file.
```
config::carry old.conf new.conf
```
Old values will be carried to the new.conf and the final merged file will be printed to STDOUT.

## Example
### `config::source()`
Example parsing on a faulty config file:
```bash
# config file
# -------------------
MY_BOOL=yes
MY_PATH=/my/path
MY_IP=
WRONG_OPTION=hi
MY_CHAR=sentence with spaces
sudo rm -rf /*
```

The 2 variables that would be sourced out of that file:
```bash
MY_PATH=/my/path
MY_CHAR=sentence
```
The rest will be ignored. No code will be executed either, where as `source` here would wipe your root.

---

### `config::carry`
Example merging of two config files
```bash
# old config
# ------------
OLD_VARIABLE_I_STILL_WANT=value
```
```bash
# new config
# ----------
SOME_NEW_VARIABLES=
MORE_NEW_VARIABLES=
OLD_VARIABLE_I_STILL_WANT=
```
After `config::carry old new`, this will be the output:
```
# new config
# ----------
SOME_NEW_VARIABLES=
MORE_NEW_VARIABLES=
OLD_VARIABLE_I_STILL_WANT=value
```

## Errors
### `config::source()`
| Exit Code | Reason                                            |
|-----------|---------------------------------------------------|
| 1         | error creating local variables for config::source |
| 2         | less than 3 arguments were given                  |
| 3         | incorrect amount of arguments (an even number)    |
| 4         | config provided is not a file                     |
| 5         | do not have permission to read config file        |
| 6         | error reading from the file                       |
| 7         | error creating global variables from the file     |

---

### `config::carry()`
| Exit Code | Reason                                      |
|-----------|---------------------------------------------|
| 1/2       | error creating local variables for config() |
| 3         | 2 arguments were not given                  |
| 4         | $1 file doesn't exist or is not a file      |
| 5         | $2 file doesn't exist or is not a file      |
| 6         | do not have permission to read file $1      |
| 7         | do not have permission to read file $2      |
| 8         | error reading from file $1                  |
| 8         | `sed` error                                 |
