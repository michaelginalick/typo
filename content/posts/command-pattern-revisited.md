---
title: 'Command Pattern Revisited'
date: "2025-12-09T16:34:06-06:00"
summary: "An update to a previous post"
description: "Classic design pattern in Go"
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "design patterns", "Command Pattern", "first class functions"]
showTags: true
hideBackToTop: false
---

Turns out Go developers don't like the classic design patterns presented in the [Gang of Four](https://en.wikipedia.org/wiki/Design_Patterns). While I'm not positive, I think it's because of the full name: "Design Patterns: Elements of Reusable Object-Oriented Software". The "Object-Oriented" part is what gives some Go developers a hard pass. Go is not object oriented. That much is difficult to dispute. It does however present users with object oriented constructs such as this:
```golang

type User struct {
    FirstName string
}

func (u *User) ModifyFirstName(firstName string) {
    u.FirstName = firstName
}

```

Or for example you can create a getter function (although its not necessary (or even common)) like so
```golang

func (u User) ReadFirstName(firstName string) string {
    return u.FirstName
}

```
Notice the lack of a pointer in the signature. This tells the reader that this method does not modify the instance. Reguardless, User is not a class. Getting an instance of it doesn't require a specific constructor. You can create one like this:

```golang
    u := &User{}
```

or like this:

```golang
    u : &User{FirstName: "Mike"}
```

You can even do this:

```golang
    func NewUser(fName string) *User {
        return &User{
            FirstName: fname
        }
    }
```

These are all functionality equivilant. Note there is a `new` keyword, but it is rarely used and is syntatic sugar. These are the same.
```golang
    u := new(User)
    uu := &User{}
```

The point is: Go is not object-oriented, but that doesn’t mean anything from OOP is bad, useless or doesn't belong in Go. More importantly, it doesn’t mean that concepts from other paradigms have nothing to teach. Design patterns are and will always be extremely valuable — if nothing else, they provide a shared vocabulary for discussing solutions. I’m not saying you should force every problem into a pattern. But having a conceptual understanding of common patterns is useful.

This common language came up recently on a project at work. I will spare a lot of the details since they won't make any sense to anyone not familiar with this exact system. At a high level, there is of files. You can think of them like rules and you want to run a process against a specific set of rules. However, all the rules need to be backwards compatible so if you create a version 2 of an existing rule set, version 2 must be able to execute version 1. In the end anytime you create a new set of these versioned rules, you basically create a new file, copy and paste the existing version of the rule set into the new file, then add whatever changes you need in this new versioned file. One caveat is that in the individual rule files, there are unique identifiers, and they often refer to that particular version. When creating a new version you must go through and update these identifiers to be the new version. Make sense? Probably not, but that's ok. 

After creating a few of these new file versions a team member suggested we make a tool that does this for us. Great idea! 
The first version was pretty simple, but as more functionality was added the code became a bit cumbersome and difficult to understand. A refactor was in order and I had an idea on how things could be organized.

The tool was running a series of steps. If any one step failed, then the program would exit. It didn't need to concern itself with things like cleanup. To me, this was a use case for a flavor of the command pattern. The implementation was fairly straightforward. It looks something like this:

```golang

type Action int

const (
	ReadInFile Action = iota
	TransFormContents
	WriteNewFile
)

type Command struct {
	action Action
}

type Commands struct {
	commands []Command
}

type FileModificationCommand struct {
	Commands
	name string
}

func (f *FileModificationCommand) Call() error {
	for _, cmd := range f.commands {
		cmd.Call()
	}
	return nil
}

func NewFileCommand(action Action) Command {
	return Command{action: action}
}

func (c *Command) Read() {
	fmt.Println("Read")
}

func (c *Command) TransFormContents() {
	fmt.Println("Transform")
}

func (c *Command) Write() {
	fmt.Println("Write")
}

func (c *Command) Call() {
	switch c.action {
	case ReadInFile:
		c.Read()
	case TransFormContents:
		c.TransFormContents()
	case WriteNewFile:
		c.Write()
	}
}

func NewCommand() *FileModificationCommand {
	fc := &FileModificationCommand{}

	fc.commands = append(fc.commands,
		NewFileCommand(ReadInFile),
		NewFileCommand(TransFormContents),
		NewFileCommand(WriteNewFile),
	)

	return fc
}

func main() {
	f := NewCommand()
	f.Call()
}
```
This is a high level version of what was submitted. To be completely honest I was feeling pretty good about how it turned out. While I didn't explicitly say I used the command pattern, it was pretty evident from the naming convention.

The implementation generated many comments, some negative, revolving mostly around how this implementation is too OOP and not idiomatic Go. Other comments were much more helpful. These comments took note of all the boiler plate - specifically these sections:
```golang

type Action int

const (
    ReadInFile Action = iota
    TransFormContents
    WriteNewFile
)

func (c *Command) Call() {
    switch c.action {
        case ReadInFile:
            c.Read()
        case TransFormContents:
            c.Transform()
        case Write:
            c.Write()
    }
}

```

The primary issue the commenter called out was the `Action` (Go's version of an Emun) and `Call` function. This function looks at the command's `Action` and calls the appropriate method. This implementation is completely unnecessary since Go has first class functions. Instead of associating an `Action` with a method, you can simply call the function. This is great feedback. With it, the code can look something like this:

```golang

type FileModificationCommand struct {
	Commands []*Command
	name     string
}

type cmd func() error
type Command struct {
	fn cmd
}

func (f *FileModificationCommand) Call() error {
	for _, cmd := range f.Commands {
		if err := cmd.fn(); err != nil {
			return err
		}
	}
	return nil
}

func NewFileCommand(f cmd) *Command {
	return &Command{fn: f}
}

func Read() error {
	fmt.Println("READ")
	return nil
}
func TransFormContents() error {
	fmt.Println("TRANSFORM")
	return nil
}
func Write() error {
	fmt.Println("WRITE")
	return nil
}

func NewCommand() *FileModificationCommand {
	fc := &FileModificationCommand{}
	fc.Commands = append(fc.Commands, NewFileCommand(Read))
	fc.Commands = append(fc.Commands, NewFileCommand(TransFormContents))
	fc.Commands = append(fc.Commands, NewFileCommand(Write))

	return fc
}

func main() {
	f := NewCommand()
	f.Call()
}

```

This implementation, while it doesn't move away from the usage of the pattern (by design), is much more idiomatic. It doesn't require the boilerplate that decides which method to call based on a given `Action`. It doesn't require an `Action` at all. This is very good in my opinion. There is less code, and less indirection. Functions are able to be defined, passed around, and directly envoked in an extendable pattern.

Aside from the final code being an overall better implementation, what I really enjoyed was the conversation around the usage of the pattern. It appeared evident that team members who knew the pattern had a lot to say. I believe that is because we were all converging around a common language. They knew the pattern, saw what I was trying to do, and offered great feedback and a better way to do it.

People argue all the time about these kinds of classical design patterns and whether or not they are relevant. Some claim you don't need them as they are not necessary. Personally, I think that is the wrong approach. Yes, knowing the Command Pattern is not going to get you a promotion at work or make you a 10x developer. What I believe is a better approach is to think of it in terms of knowing these patterns is not going to make you a worse developer.

I recently read a review of an album I like. In the review, the author said something along the lines of "it is clear that this band knows all the rules of music, and are deliberately breaking them." In many ways, I think of design patterns as the "rules of development". If you know them, you can decide when they should be deployed. Likewise, you also know their limitations or when no one rule will suffice. Again, this won't necessarily make you a better developer, but I am confident that this knowledge will not make you worse.