---
title:  "Serializing Java objects using Golang's gob encoding"
date:   2016-07-16 17:10:41+02:00
description: Using Golang's gob encoding in Java
---

A while ago while working with Terraform, I started reading up on Go's [RPC][go_rpc] library, and
the encoding format it uses by default, which is [gob][go_gob].
Then I found out there was no Java implementation for it. Which is a shame, because there are a few
Go programs that might be nice to interface with from the software we're building at work.
However, at work we use Java.

So I started working on an implementation, called [jgobs][jgobs]. If you can think of a better
name, please let me know.. this one isn't very original.  Anyway, I'm not even sure this is going
to be useful to us, let alone anyone else, but hey, it's a nice exercise.

Last week jgobs was able to serialize a simple (primitive fields only) object to a byte array
which was deserialized by Go's gob library.
This morning jgobs succesfully serialized an object that contains nested objects, which I think
is pretty cool, so time for a small demo!

What we'll do is serialize a simple `Person` object that has a name and an `Address` object
associated with it. Then we'll deserialize it using Go.

```java
// Address.java
package xyz.vanduuren.jgobs.demo;

public class Address {
    public String streetName;
    public int houseNumber;
    public String city;
}
```

```java
// Person.java
package xyz.vanduuren.jgobs.demo;

public class Person {
    public String name;
    public Address address;
}
```

```java
// App.java
package xyz.vanduuren.jgobs.demo;

import xyz.vanduuren.jgobs.lib.Encoder;

import java.io.FileOutputStream;
import java.io.IOException;

public class App
{
    public static void main(String[] args) throws IOException {
        // Encode to a file
        FileOutputStream fileOutputStream = new FileOutputStream("/tmp/gob_person.data");
        // Create the encoder with the file outputstream
        Encoder encoder = new Encoder(fileOutputStream);

        // Create the person and address objects we want to encode
        Person person = new Person();
        Address address = new Address();

        // Give them some data
        address.streetName = "Kerksteeg";
        address.city = "Leiden";
        address.houseNumber = 1;
        person.name = "Jan Klaassen";
        person.address = address;

        // Encode, and close the stream
        encoder.encode(person);
        fileOutputStream.close();
    }
}
```

Running the above program will result in a `Person` named Jan Klaassen that will
be serialized to `/tmp/gob_person.data`.
We should be able to decode that with a simple Go program. Let's try.


```go
// decode.go
package main

import (
    "encoding/gob"
    "fmt"
    "os"
)

func main() {
    type Address struct {
	StreetName  string
	HouseNumber int8
	City        string
    }

    type Person struct {
	Name    string
	Address Address
    }

    f, err := os.Open("/tmp/gob_person.data")
    if err != nil {
	fmt.Printf("open: %v\n", err)
	os.Exit(1)
    }

    dec := gob.NewDecoder(f)
    var person Person
    err = dec.Decode(&person)
    if err != nil {
	fmt.Printf("decode: %v\n", err)
	os.Exit(1)
    }

    fmt.Printf("decoded person: %v\n", person)
}
```

Let's compile and run it.

```bash
$ go build decode.go && ./decode

decoded person: {Jan Klaassen {Kerksteeg 1 Leiden}}
```

And it worked!
There's still a lot more stuff to do for the encoder, and then I need to start
work on the decoder. But I thought this was already pretty cool! 😄

[go_rpc]:	https://golang.org/pkg/net/rpc/
[go_gob]:	https://golang.org/pkg/encoding/gob/
[jgobs]:	https://github.com/boyvanduuren/jgobs
