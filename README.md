# Writing a simple shell in Go

The readme is just a copy of [this](https://sj14.gitlab.io/post/2018-07-01-go-unix-shell/) blog post.

# Introduction

In this post, we will write a minimalistic shell for UNIX(-like) operating systems in the Go programming language and it only takes about 60 lines of code. You should be a little bit familiar with Go (e.g. how to build a simple project) and the basic usage of a UNIX shell.

> UNIX is very simple, it just needs a genius to understand its simplicity. - [Dennis Ritchie](https://en.wikipedia.org/wiki/Dennis_Ritchie)

Of course, I'm not a genius, and I'm not even sure if Dennis Ritchie meant to include the userspace tools. Furthermore, a shell is only one small part (and compared to the kernel, it's _really_ an easy part) of a fully functional operating system, but I hope at the end of this post, you are just as astonished as I was, how simple it is to write a shell once you understood the concepts behind it.

# What is a shell?

Definitions are always difficult. I would define a shell as the basic user interface to your operating system. You can input commands to the shell and receive the corresponding output. When you need more information or a better definition, you can look up the Wikipedia [article](https://en.wikipedia.org/wiki/Shell_(computing)).

Some examples of shells are:

- [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell))
- [Zsh](https://en.wikipedia.org/wiki/Z_shell)
- [Gnome Shell](https://en.wikipedia.org/wiki/GNOME_Shell)
- [Windows Shell](https://en.wikipedia.org/wiki/Windows_shell)

The graphical user interfaces of Gnome and Windows are shells but most IT related people (or at least I) will refer to a text-based one when talking about shells, e.g. the first two in this list. Of course, this example will describe a simple and non-graphical shell.

In fact, the functionality is explained as: give an input command and receive the output of this command. An example? Run the program `ls` to list the content of a directory.

Input:
```text
ls
```

Output:
```text
Applications			etc
Library				home
...
```

That's it, super simple. Let's start!

# The input loop

To execute a command, we have to accept inputs. These inputs are done by us, humans, using keyboards ;-) 

The keyboard is our standard input device (`os.Stdin`) and we can create a reader to access it. Each time, we press the enter key, a new line is created. The new line is denoted by `\n`. While pressing the enter key, everything written is stored in the variable `input`.

```go
reader := bufio.NewReader(os.Stdin)
input, err := reader.ReadString('\n')
```

Let's put this in a `main` function and adding a `for` loop around the `ReadString`. When an error occurs while reading the input, we will just print it.  
 
 Side note: The error is printed to the standard output device (`os.Stdout`). It would be better to write errors to the standard error device (`os.Sterr`), e.g. by using `log.Println`, but this will add a timestamp to the output and look too verbose. Of course, you can change it later.

```go
func main() {
	reader := bufio.NewReader(os.Stdin)
	for {
		// Read the keyboad input.
		input, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println(err)
		}
	}
}
```

# Executing Commands

Now, we want to execute the entered command. Let's create a new function `execInput` for this.  First, we have to remove the newline control character `\n` at the end of the input. Next, we can prepare the command with `exec.Command(input)` and execute it with `cmd.CombinedOutput()`.  
When the execution was successful, the output would be written to `stdout`, and when there was an error, the output would be written to `stderr`. In both ways, we save the result in the variable `stdoutStderr` and as the last step, we print whatever has been stored there (this will use `stdout` only).

```go
func execInput(input string) error {
	// Remove the newline character.
	input = strings.TrimSuffix(input, "\n")

	// Prepare the command to execute.
	cmd := exec.Command(input)

	// Execute the command and save it's output.
	stdoutStderr, err := cmd.CombinedOutput()
	if err != nil {
		return err
	}

	// Print the output.
	fmt.Printf("%s", stdoutStderr)
}
```

# First Prototype

We complete our `main` function by adding a fancy input indicator (`>`) at the top of the loop, and by adding the new `execInput` function at the bottom of the loop.

```go
func main() {
	reader := bufio.NewReader(os.Stdin)
	for {
		fmt.Print("> ")
		// Read the keyboad input.
		input, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println(err)
		}

		// Handle the execution of the input.
		err = execInput(input)
		if err != nil {
			fmt.Println(err)
		}
	}
}
```

It's time for a first test run. Build and run our shell with `go run main.go`. You should see the input indicator `>` and be able to write something. For example, we could run the `ls` command.  

```text
> ls
LICENSE
main.go
main_test.go
```

Wow, it works! The program `ls` was executed and gave us the content of the current directory. You can exit the shell just like most other programs with the key combination `CTRL-C`.

# Arguments

Let's get the list in long format with `ls -l`.

```text
> ls -l
exec: "ls -l": executable file not found in $PATH
```

It's not working anymore. This is because our shell tries to run the program `ls -l`, which is not found. The program is just `ls` and `-l` is a so-called argument, which is parsed by the program itself. Currently, we don't distinguish between the command and the arguments. To fix this, we have to modify the `execLine` function and split the input on each space.

```go
func execInput(input string) error {
	// Remove the newline character.
	input = strings.TrimSuffix(input, "\n")

	// Split the input to separate the command and the arguments.
	args := strings.Split(input, " ")

	// Pass the program and the arguments separately.
	cmd := exec.Command(args[0], args[1:]...)
	...
}
```

The program name is now stored in `args[0]` and the arguments in the subsequent indexes. Running `ls -l` now works as expected.

```text
> ls -l
total 24
-rw-r--r--  1 simon  staff  1076 30 Jun 09:49 LICENSE
-rw-r--r--  1 simon  staff  1058 30 Jun 10:10 main.go
-rw-r--r--  1 simon  staff   897 30 Jun 09:49 main_test.go
```

# Change Directory (cd)

Now we are able to run commands with an arbitrary number of arguments. To have a set of functionality which is necessary for a minimal usability, there is only one thing left (at least according to my opinion). You might already come across this while playing with the shell: you can't change the directory with the `cd` command.

```text
> cd /
> ls
LICENSE
main.go
main_test.go
```

No, this is definitely not the content of my root directory. Why does the `cd` command not work? When you know, it's easy: there is no [_real_](https://stackoverflow.com/a/38776411) `cd` program, the functionality is a built-in command of the shell.

Again, we have to modify the `execInput` function. Just after the `Split` function, we add a `switch` statement on the first argument (the command to execute) which is stored in `args[0]`. When the command is `cd`, we check if there are subsequent arguments, otherwise, we can not change to a not given directory (in most other shells, you would then change to your home directory). When there is a subsequent argument in `args[1]` (which stores the path), we change the directory with `os.Chdir(args[1])`. At the end of case block, we return the `execInput` function to stop further processing of this built-in command.  
Because it is so simple, we will just add a built-in `exit`function right below the `cd` block, which stops our shell (an alternative to using `CTRL-C`).

```go
// Split the input to separate the command and the arguments.
args := strings.Split(input, " ")

// Check for built-in commands.
switch args[0] {
case "cd":
	// 'cd' to home dir with empty path not yet supported.
	if len(args) < 2 {
		return  errors.New("path required")
	}
	err := os.Chdir(args[1])
	if err != nil {
		return err
	}
	// Stop further processing.
	return nil
case "exit":
	os.Exit(0)
}
...
```

Yes, the following output looks more like my root directory.

```text
> cd /
> ls
Applications
Library
Network
System
...
```

That's it. We have written a simple shell :-)

# Considered improvements

When you are not already bored by this, you can try to improve your shell. Here is some inspiration:

- Modify the input indicator:
  - add the working directory
  - add the machine's hostname
  - add the current user
- Write errors to `stderr`
- Browse your input history with the up/down keys.

# Conclusion

We reached the end of this post and I hope you enjoyed it. I think, when you understand the concepts behind it, it's quite simple.  

Go is also one of the more simple programming languages, which helped us to get to the results faster. We didn't have to do any low-level stuff as managing the memory ourselves. Rob Pike and Ken Thompson, who created Go together with Robert Griesemer, also worked on the creation of UNIX, so I think writing a shell in Go is a nice combination.

As I'm always learning too, please just contact me whenever you find something which should be improved.
