---
title: "Chain of Responsibility"
date: 2024-11-08T11:06:37-06:00
summary: "Utilizing established design patterns"
description: "Utilizing established design patterns"
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "design patterns", "chain of responsibility"]
showTags: true
hideBackToTop: false
---

All software projects evolve over time. Requirements change, and new functionality is often added, updated, or removed. If the project you're working on isn’t experiencing these changes, it could be a good sign or a warning sign.

The project I’m currently working on is experiencing just such evolution. At a high level, it involves a piece of data existing in one system. When that data is modified, the system sends the updated data to another system for processing. This second system is responsible for, among other things, applying a set of modifications to the data and then persisting it in the data layer. For all intents and purposes, this is an ETL (Extract, Transform, Load) process.

The "T" in ETL—the transformation stage—has evolved over time. Initially, all data underwent the same set of transformations, so there wasn’t much complexity to manage. Regardless of the content, the data always behaved in the same way, so much so that the modifications existed within the persistence package. (This is not ideal, I know!) But, as is often the case, requirements changed. Now, the data being processed needs to specify the transformations it requires. Data type A must behave differently from Data type B, which behaves differently from Data type C, and so on. Clearly, this posed a problem—the system wasn't designed for this level of flexibility.

I’ll spare you the detailed refactoring process, but in essence, the new requirements could be summarized as follows:

- Each data type needs to specify the transformations it requires.
- An arbitrary number of transformations may be applied.
- Transformation functions should be able to call the next transformation function in sequence.

Like most software challenges, this one had been solved before and not only solved but solved frequently enough to have an established pattern: the Chain of Responsibility. What a relief! While the following example is a more simplified version, the fundamental structure is the same. It’s remarkable what can be achieved with a pointer and a linked list.


```golang

type Modifier interface {
	Add(m Modifier)
	Handle()
}

type Person struct {
	Age int
}

type PersonModifier struct {
	p    *Person
	next Modifier
}

func NewPerson(age int) *Person {
	return &Person{Age: age}
}

func NewPersonModifier(person *Person) *PersonModifier {
	return &PersonModifier{p: person}
}

type DoubleAgeModifier struct {
	PersonModifier
}

func NewAgeModifier(p *Person) *DoubleAgeModifier {
	return &DoubleAgeModifier{PersonModifier{p: p}}
}

func (p *PersonModifier) Add(m Modifier) {
	if p.next != nil {
		p.next.Add(m)
	} else {
		p.next = m
	}
}

func (c *PersonModifier) Handle() {
	if c.next != nil {
		c.next.Handle()
	}
}

func (m DoubleAgeModifier) Handle() {
	m.p.Age *= 2

	m.PersonModifier.Handle()
}

func main() {
  person := NewPerson(10)

  root := NewPersonModifier(person)
  root.Add(NewAgeModifier(person))
  root.Add(NewAgeModifier(person))

  root.Handle()

  fmt.Println("Age %d", person.Age)
}

```

With this simple construct, modification functions can be added and chained with just a few lines of code.
