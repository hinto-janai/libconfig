# libconfig
>Function for sourcing files safely in Bash

## Contents
- [About](#About)
- [Speed](#Speed)
- [Usage](#Usage)
- [Input & Data Types](#InputDataTypes)
- [Valid Input](#ValidInput)
- [Examples](#Examples)
- [Errors](#Errors)

## About
This function parses and sources a text file without executing the code from the file itself, like the very unsafe `source` does. 100% Bash builtins, no external binaries called. Requires Bash v5+.

## Speed
`config()` scales around ***2-40x~ more poorly*** than `source` as the number of lines in the config file increase, and as the arguments you're looking for increases.

The nominal time difference however, is small. Using `config()` vs `source` on 100 lines will be an un-noticable difference.

| lines of text | source time | config() time |
|---------------|-------------|---------------|
| 10            | 0.000s      | 0.001s        | 
| 100           | 0.001s      | 0.010s        |
| 1000          | 0.002s      | 0.080s        |

## Usage
Copy the `config()` function from `config.sh` into your script [or preferably, use hbc to "compile" this library together with your script, click here for more details.](https://github.com/hinto-janaiyo/hbc)

## Input & Data Types
`config()` needs at the minimum 3 arguments:
- `CONFIG_PATH`
- `DATA_TYPE`
- `OPTION_NAME`

With every 2 pair argument after that being another option you'd like to read from the same config file.

Here's a typical configuration file you'd like to read:
```bash
MY_IP=127.0.0.1
MY_INT=4
MY_PORT=127.0.0.1:80
MY_BOOL=true
MY_CHAR=hello
MY_PATH=/home/hinto
```

To parse this with `config()`, it would look like:
```bash
config /path/to/config \
	ip   MY_IP   \
	int  MY_INT  \
	port MY_PORT \
	bool MY_BOOL \
	char MY_CHAR \
	path MY_PATH
```
`config()` will read the file provided line-by-line and look for `OPTION_NAME=value`. If the value is empty, or does not match the data type provided (MY_IP=not_an_ip), then it will not intialize that option as a variable.

If it matches (MY_BOOL=true), it will intialize it as a global variable: `declare -g MY_BOOL=true`

## Valid Input
| Data Type | Valid Input                                                  | [[ Bash =~ Regex Pattern ]]      |
|-----------|--------------------------------------------------------------|----------------------------------|
| ip        | integers seperated by '.'; must begin/end with integer       | ^${2}=[0-9.]+'.'[0-9]+$          |
| int       | any positive integer including 0; no floating point          | ^${2}=[0-9]+$                    |
| port      | like ip, but expects a port ':'; must begin/end with integer | ^${2}=[0-9:.]+'.'[0-9]':'[0-9]+$ |
| bool      | true/false                                                   | ^${2}=true$ OR ^${2}=false$      |
| char      | alphanumeric and some extra characters                       | ^${2}=[[:alnum:]._-]+$           |
| path      | alphanumeric seperated by '/'; no spaces allowed             | ^${2}=[[:alnum:]./_-]+$          |

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

## Example
Example parsing by `config()` on a faulty config file:
```bash
# example config file
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

## Errors
| Exit Code | Reason                                           |
|-----------|--------------------------------------------------|
| 1         | error creating local variables for config()      |
| 2         | less than 3 arguments were given                 |
| 3         | incorrect amount of arguments (an even number)   |
| 4         | config provided is not a file                    |
| 5         | do not have permission to read config file       |
| 6         | error reading from the file                      |
| 7         | error creating global variables from the file    |
