---
title: "Command Pattern"
date: 2025-01-15T11:41:00-06:00
summary: "Using a simple pattern to coordinate actions"
description: "Using a simple pattern to coordinate actions"
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "design patterns", "Command Pattern"]
showTags: true
hideBackToTop: false
---


The [Command Pattern](https://en.wikipedia.org/wiki/Command_pattern) is a well established design pattern. It has lots of use cases and the implementation is flexible is useful is many different situations. At a previous company, there were many proponents of an excellent ruby library called, [light-service](https://github.com/adomokos/light-service). If you look closely enough, this is one example implementation of the pattern.

Generally speaking, any program you write is a command. Sometimes you want to do a simple thing like update a record in a database, and other times you need to coordinate a set of changes across many different services. The pattern lends itself to the latter use case. I was recently tasked with doing some coordinated updates, in which I found the command pattern to be very useful.

I've written previously about how much of my work involves using Lambda functions to consume events from an event bus. In this case, it is AWS EventBridge, but it could be swapped out for Kafka or even a queue. Lambdas are nice because, almost by definition, they are small, focused programs. They do one or two things without too much knowledge of the outside world. One of the higher-level patterns in the organization I work for is the concept of the write-back. Not exactly in the sense of caching, but in some ways, yes. I manage many different indices in OpenSearch, but only a few of them are exposed to outside clients. However, data needs to be synced across the exposed indices so clients see updated data.

The data team at my current job runs a lot of processes that the indices I manage care about. When one such process completes, it places an event on the bridge, which subsequently invokes a Lambda to pull from one data store (in this case, Snowflake) and ingest the data into the correct OpenSearch index. Once the ingestion process is complete, the Lambda then syncs all the new data with current records in another index. At a high level, this program needs to:

1. Pull data into a new Opensearch index (with the same alias).
2. Swap the new and old indices.
3. Process all the records across the external incides. This is basically a way to write the new data onto the correspnding document in another location.

As this process ran in production over the course of a few weeks, we started to see CPU spikes on the OpenSearch cluster. After some digging, we found that these spikes happen almost exactly when the "write-back" process runs. After even more digging, I found this [blog post](https://opensearch.org/blog/optimize-refresh-interval/) about refresh intervals and how they are used in Opensearch.

OpenSearch is constantly running processes to sync data across all the nodes in the cluster. Because it uses an eventual consistency model, changes percolate through the nodes; they don't happen all at once like in a traditional RDB. This means when you add data to an index, changes will appear as quickly as possible throughout the entire system. However, these processes are not free. If you are making large sets of changes and have many clients asking for data, OpenSearch needs to manage all of this. As such, the CPU spikes during these update periods are due to OpenSearch syncing all the nodes in the cluster. To help alleviate these spikes, we want to update the refresh interval to be every 60 seconds during ingest periods, then when the process is complete, set it back to every 1 second.

Ok. So now the program needs to do a few more things

1. Pull data into a new Opensearch index (with the same alias).
2. Swap the new and old indices.
3. Set the refresh interval to 60 seconds on the index being written to.
4. Process all the records across the external incides.
5. Set the refresh interval back to 1 second when the records have been updated.

One more caveat to this, is that if any one of these operations fail, we need to

1. Ensure any new indices are deleted. Effectively rollback to the previous index.
2. Set the refresh interval back to 1 second.

After updating the requirements for this Lambda, the use case for the command pattern became clearer to me. Thankfully, Go makes it very easy to implement these kinds of patterns. Below I will show the outline I used to implement the Command Pattern using the example of a money transfer from one account to another.

The first thing was to develop a contract in the form of an interface.

```golang
type Command interface {
  Call()
  Succeeded() bool
  SetSucceeded(value bool)
  Undo()
}
```

You don't typically see a lot of getters and setters in Go, but this example aims for clarity and not necessarily the most idiomatic code. This interface is very simple; it basically has a Call method that will be in charge of invoking all the individual commands, two methods for determining the result of individual commands, and an Undo method. This should be called if any one operation fails.

Now that we have a contract, there needs to be a way to aggregate this to accommodate many commands. Fortunately, we can embed interfaces in structs.

```golang
type CompositeCommand struct {
  commands []Command
}
```

Now that there is a contract via interface and a way to aggregate the interface, we can add some additional structs, some of which will implement the Command interface. For this, struct composition will be used, which is a great way to model data in Go. Using this technique, we can satisfy the Command interface without directly implementing it on every struct type.

The first is a simple Account struct with a balance field. It will also have two methods for adding and subtracting.

```golang

type BankAccount struct {
	balance int
}

func (b *BankAccount) Deposit(amount int) {
	b.balance += amount
	fmt.Println("Deposited", amount, "\n, balance is now", b.balance)
}

func (b *BankAccount) Withdraw(amount int) {
	b.balance -= amount
	fmt.Println("Withdrew", amount, "\n, balance is now", b.balance)
}

```

Now a command can be introduced that acts on an account. This struct will implement the Command interface.

```golang
type Action int

const (
	Deposit Action = iota
	Withdraw
)

type BankAccountCommand struct {
	account   *BankAccount
	action    Action
	amount    int
	succeeded bool
}

func (b *BankAccountCommand) SetSucceeded(value bool) {
	b.succeeded = value
}

func (b *BankAccountCommand) Succeeded() bool {
	return b.succeeded
}

func (b *BankAccountCommand) Call() {
	switch b.action {
	case Deposit:
		b.account.Deposit(b.amount)
		b.succeeded = true
	case Withdraw:
		b.account.Withdraw(b.amount)
    b.succeeded = true
	}
}

func (b *BankAccountCommand) Undo() {
	fmt.Println("UNDO!")
}

func NewBankAccountCommand(account *BankAccount, action Action, amount int) *BankAccountCommand {
	return &BankAccountCommand{account: account, action: action, amount: amount}
}


```
With this in place, there is a struct that manages itself, and one that manages the individual actions. This is where the CompositeCommand will come into play. By implementing the Command interface on the CompositeCommand struct, it allows for acting on multiple commands.

```golang
type CompositeCommand struct {
  commands []Command
}

func (c *CompositeCommand) Succeeded() bool {
	for _, cmd := range c.commands {
		if !cmd.Succeeded() {
			return false
		}
	}
	return true
}

func (c *CompositeCommand) SetSucceeded(value bool) {
	for _, cmd := range c.commands {
		cmd.SetSucceeded(value)
	}
}

func (c *CompositeCommand) Call() {
	for _, cmd := range c.commands {
		cmd.Call()
	}
}

func (c *CompositeCommand) Undo() {
	fmt.Println("UNDO!")
}
```

Finally, we can use the composite command and tie everything together by composing a transfer struct.

```golang
type TransferCommand struct {
	CompositeCommand
	from, to *BankAccount
	amount   int
}

func (m *TransferCommand) Call() {
	ok := true
	for _, cmd := range m.commands {
		if ok {
			cmd.Call()
			ok = cmd.Succeeded()
		} else {
			cmd.SetSucceeded(false)
		}
	}
}

func NewTransferCommand(from *BankAccount, to *BankAccount, amount int) *TransferCommand {
	c := &TransferCommand{from: from, to: to, amount: amount}
	c.commands = append(c.commands, NewBankAccountCommand(from, Withdraw, amount))
	c.commands = append(c.commands, NewBankAccountCommand(to, Deposit, amount))
	return c
}
```
If you're interested in running this code, you can do so [here](https://go.dev/play/p/A4QVxygpRyl). lthough this is a simple example, it employs powerful techniques such as struct and interface composition. These patterns help simplify a lot of the code I was struggling with as more and more requirements emerged.
