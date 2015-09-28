# Structuring Bash scripts

## Commenting the code

Bash has no native support for mulitline comments. Instead, the following repurposing of ```here documents``` is used - with all the caveats that errors in its construction can leat ot unexpected (and potentially danerous) side affects.

``` shell
# Example of a multi-line comment
: <<'EOF'
This is line one of a multi-line comment;
This is line two of a multi-line commect;
'EOF'
``` shell

Note the following in its use:

1. ```:``` is a shell built in null command (NOP) that does nothing;
2. ```EOF``` is the word used to delimit the ```here document```. *** It must be quoted *** otherwise the content of the here-document will be expanded.

## Redirectoring stdout and stderr

The following are equivalent and redirect both stdout and stderr to the null device:

``` shell
some_command  > /dev/null 2>&1
```

``` shell
some_command &> /dev/null
``` shell

## Conditionally executing command(s)

For example, installing ```Homebrew``` if it is not installed already:

``` shell
command -v brew &> /dev/null || {
	echo "Installing Homebrew ..."
	# Install Homebrew 
}
```

Note the following:

1. ```command -v``` describes a command without executing if, exiting with 1 if it doesn't exist;
2. ```||``` executes the command on the rhs if the command on the lhs returns with exit 1;
3. ``` { ... }``` is a ```multi-line command group``` - use ```;``` to delimit commands if all on a single line;

## Functions

- The ```function``` keyword is optional unless the function name is also an alias;
 - If the ```function``` keyword is used, parantheses are then optional;

### Execution scope

Functions can be declared to run in the current shell using curly braces to scope the body.

``` shell
my_func () {
	# Executes within the current environment
}
```

Whereas functions declared using brackets run in a new subshell.

``` shell
my_func () (
	# Executes within a new shell
)
```

### Forward declaration

Functions need to be defined before they are used. This is commonly achieved using the following pattern:

``` shell
main () {
	foo
}

foo () {
}

# $@ is the array of all positional parameters $1, $2, etc
main "$@"
```

### Functions global to shell and all subshells

Functions can be exported to make them available to the current shell and all subshells. 

In which case:

1. Name the function sensibly 
2. Mark the function ```readonly``` to prevent it being redeclared. Note, this does not prevent it being ```unset```.

``` shell
my_func() {
}

export -f my_func && readonly -f my_func
```
### Private functions

Functions that are private to a script should be named with a ``__`` prefix and ```unset``` at the end of the script.

``` shell
__my_func () {
}

unset -f __my_func

## Bash variable scoping

### Variables global to shell and all subshells

Name the variable in upper case and using underscores.

``` shell
export MY_VARIABLE
```

### Variables local within a shell script

Variables declared with the ```local``` keyword are local to the declaring function **and all functions called from that function**.

``` shell
child_function () {
        echo "$foo"
        local foo=2
        echo "$foo"
}

main () {
        local foo=1
        echo "$foo"
        child_function
        echo "$foo"
}

main

# Actual output
# 1
# 1
# 2
# 1
```

### Variables global within a shell script

Two mechanisms can be used to prevent variables declared within a script from leaking into the calling shell environment.

1. ```unset``` each variable at the end of the script

	``` shell
	g_variable=something
	...
	unset g_variable
	```


2. Declare each variable from within a subshell scoped ```main```

	``` shell
	# Scope to subshell
	main () (
		# Global to current subshell
		g_variable=something
	)
	```

## Environment scoping

### Querying variables and functions in scope

``` shell
# List all functions declared within the current environment
compgen -A function
```

``` shell
# List all variables declared wtihin the current environment
compgen -v
```

### Excuting versus sourcing a script

When you ```execute``` a shell script, the script runs in a new shell. Any changes to the environment caused by the script are scoped to the new shell.

``` shell
./my-script.sh
```

When you ```source``` a shell script, the script runs in the current shell. Any changes to the environment caused by the script will affect the current shell.

``` shell
source my-script.sh
```

## xargs

### Running command once per item in array

``` shell
declare -a ITEMS=(item1 item2 item2)
# echo "${DEPENDS_ON[@]}" | xargs -0 -t -n1 -I {} . "${CWD}/{}"
```




