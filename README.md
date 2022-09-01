# libconfig
## Contents
- [About](#About)
- [Speed](#Speed)
- [Usage](#Input)
	- [config::grep](#ConfigGrep)
		- [Data Types](#Data-Types)
		- [Output Types](#Output-Types)
	- [config::merge](#ConfigCarry)
- [Example](#Example)
	- [config::grep](#ConfigGrep-1)
	- [config::merge](#ConfigCarry-1)
- [Errors](#Errors)
	- [config::grep](#ConfigGrep-2)
	- [config::merge](#ConfigCarry-2)

## About
These functions handle common config file operations in Bash.

| Function           | Purpose | Dependency |
|--------------------|---------|------------|
| `config::grep()`   | Parses and prints values from a text file without executing the code from the file itself, like the unsafe `source` does | `Bash v4.4+` |
| `config::merge()`  | Carrys old variable values from one file to another and prints the merged config file to STDOUT | `sed`, `grep` |

## Speed
Despite the name, `config::grep()` does not use `grep`, it is 100% Bash builtins.

| Lines of Text | Source Time | config::grep() Time | config::merge() Time |
|---------------|-------------|---------------------|----------------------|
| 10            | 0.000s      | 0.001s              | 0.002s
| 100           | 0.001s      | 0.005s              | 0.010s
| 1000          | 0.002s      | 0.030s              | 0.045s

## Usage
### `config::grep()`
Needs at the minimum 4 arguments:
- `OUTPUT_TYPE`
- `CONFIG_PATH`
- `DATA_TYPE`
- `OPTION_NAME`
Example:
```
config::grep --prefix="my_" /my/path \
	data_type  MY_OPTION_NAME \
	more_types MORE_OPTIONS
```
The order of the `CONFIG_PATH` & `OUTPUT_TYPE` don't matter, but the `DATA_TYPE` -> `OPTION_NAME` does, it must always be the type first, then the option name.

Every 2 pair argument after will be more options you'd like to read from the same config file.

### Data Types
| Data Type   | Valid Input                                                     | Example                              | Special Exception |
|-------------|-----------------------------------------------------------------|--------------------------------------|-------------------|
| ip          | alphanumeric seperated by '.'; must begin/end with alphanumeric | 127.0.0.1, my.domain.com             | localhost         |
| port        | like ip, but expects a port ':'; must end with integer          | 127.0.0.1:8080, my.domain.com:22     | localhost[:port]  |
| int         | any integer including 0 and negative; no floating point         | 88, 0, -1                            |                   |
| pos         | postive integer including 0; no floating point                  | 88, 0, 1                             |                   |
| neg         | negative integer; no floating point                             | -88, -2, -1                          |                   |
| bool        | true/false                                                      | true, false                          |                   |
| char        | alphanumeric and some extra characters                          | hi, hi_all, h1.every-one             |                   |
| path        | alphanumeric seperated by '/'; no spaces, must be full path     | /home/hinto                          |                   |
| proto       | protocol followed by '://ip' (and optionally a port)            | ssh://site.com:22, smb://127.88:137  |                   |
| web         | like protocol, but only for http, https, www                    | https://site.com/search?query=...    |                   |
| [...] (...) | custom regex range; parenthesis must be quoted: '(...)'         | [0-9], [A-Z]+, '([1-3]\|[7-9])'      |                   |

### Output Types
| Output Type | Command Flag   | Behavior                                                                       | Example                                   |
|-------------|----------------|--------------------------------------------------------------------------------|-------------------------------------------|
| prefix      | `--prefix=...` | Prints OPTION=VALUES with the `prefix` appended at the beginning of the OPTION | `--prefix=hello_` -> `hello_MY_BOOL=true` |
| map         | `--map=...`    | Prints OPTION=VALUES with the OPTION being the key to the `map` array created  | `--map=hello` -> `hello[MY_BOOL]=true`    |
| ...         | ...            | If no `--flag` is given, the raw values will be printed                        | `MY_BOOL=true`

Here's a typical configuration file you'd like to read:
```bash
MY_IP=127.0.0.1
MY_INT=4
MY_PORT=127.0.0.1:80
MY_BOOL=true
MY_CHAR=hello
MY_PATH=/home/hinto
```

To parse this with `config::grep()`, it would look like:
```bash
config::grep --map=my_array /path/to/config \
	ip   MY_IP   \
	int  MY_INT  \
	port MY_PORT \
	bool MY_BOOL \
	char MY_CHAR \
	path MY_PATH
```
`config::grep()` will read the file provided line-by-line and look for `OPTION_NAME=value`. If the value is empty, or does not match the data type provided (MY_IP=not_an_ip), then it will not print that option.

If it matches (MY_BOOL=true), it will print it in the output type you specified.

If multiple arguments are given in the config file, for example:
```bash
MY_BOOL=true
MY_BOOL=false
```
Both version will be printed: `MY_BOOL=true MY_BOOL=false`

Note:
* Empty spaces will be stripped
* Variables with `dashes` will be defined with underscores due to Bash limitations: `$MY-BOOL` -> `$MY_BOOL` 
* Quote characters `"` and `'` will be stripped before parsing, so a config file can use them or not
```bash
MY_BOOL="true"
MY_BOOL='true '
MY-BOOL=true
MY_BOOL=true
```
These will all be parsed and printed as `MY_BOOL=true`

An implementation of sourcing the output of `config::grep`:
```bash
# source globally
declare -Ag $(config::grep --map=array /my/config \
	bool MY_BOOL \
	char MY_CHAR
)

# source locally
local -A $(config::grep --map=array /my/config \
	bool MY_BOOL \
	char MY_CHAR
)
```
This will define a mapped array called `array` with the key values as the OPTIONS:
```
array[MY_BOOL]=true
array[MY_CHAR]=hello
```

---

### `config::merge()`
Needs exactly 2 arguments, an old file and a new file.
```
config::merge old.conf new.conf
```
Old values will be carried to the new.conf and the final merged file will be printed to STDOUT. If a value contains spaces, it will be carried over to the new file retaining the spaces with `"` surrounding it:
```
VARIABLE_WITH_SPACES="this will stay like this"
```

## Example
### `config::grep()`
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

The 1 variable that would be sourced out of that file:
```bash
MY_PATH=/my/path
```
The rest will be ignored. No code will be executed either, where as `source` here would wipe your root.

---

### `config::merge`
Example merging of two config files
```bash
# old config
# ------------
OLD_VARIABLE_I_STILL_WANT=value
VARIABLE_WITH_SPACES="this has spaces"
```
```bash
# new config
# ----------
SOME_NEW_VARIABLES=
MORE_NEW_VARIABLES=
VARIABLE_WITH_SPACE=
OLD_VARIABLE_I_STILL_WANT=
```
After `config::merge old new`, this will be the output:
```
# new config
# ----------
SOME_NEW_VARIABLES=
MORE_NEW_VARIABLES=
VARIABLE_WITH_SPACES="this has spaces"
OLD_VARIABLE_I_STILL_WANT=value
```

## Errors
### `config::grep()`
| Exit Code | Reason                                            |
|-----------|---------------------------------------------------|
| 1         | error creating local variables                    |
| 2         | less than 3 arguments were given                  |
| 3         | incorrect amount of arguments (an even number)    |
| 4         | config provided is not a file                     |
| 5         | do not have permission to read config file        |
| 6         | error reading from the file                       |
| 7         | invalid data type given                           |
| 8         | no matches found                                  |

---

### `config::merge()`
| Exit Code | Reason                                      |
|-----------|---------------------------------------------|
| 1/2       | error creating local variables              |
| 3         | 2 arguments were not given                  |
| 4         | $1 file doesn't exist or is not a file      |
| 5         | $2 file doesn't exist or is not a file      |
| 6         | do not have permission to read file $1      |
| 7         | do not have permission to read file $2      |
| 8         | error reading from file $1                  |
| 8         | `sed` error                                 |
