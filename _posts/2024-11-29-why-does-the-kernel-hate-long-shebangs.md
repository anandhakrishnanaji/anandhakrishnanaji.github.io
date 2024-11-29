---
title: "Why Does the Kernel Hate Long Shebangs?"
date: 2024-11-29 18:03:00 +0530
categories: [Operating System]
tags: [computer-science, os]
render_with_liquid: false
image:
  path: /assets/img/headers/2024-11-29-why-does-the-kernel-hate-long-shebangs-social-preview.webp
  lqip: data:image/webp;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/2wBDAQkJCQwLDBgNDRgyIRwhMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjL/wAARCAAaAC0DASIAAhEBAxEB/8QAHwAAAQUBAQEBAQEAAAAAAAAAAAECAwQFBgcICQoL/8QAtRAAAgEDAwIEAwUFBAQAAAF9AQIDAAQRBRIhMUEGE1FhByJxFDKBkaEII0KxwRVS0fAkM2JyggkKFhcYGRolJicoKSo0NTY3ODk6Q0RFRkdISUpTVFVWV1hZWmNkZWZnaGlqc3R1dnd4eXqDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uHi4+Tl5ufo6erx8vP09fb3+Pn6/8QAHwEAAwEBAQEBAQEBAQAAAAAAAAECAwQFBgcICQoL/8QAtREAAgECBAQDBAcFBAQAAQJ3AAECAxEEBSExBhJBUQdhcRMiMoEIFEKRobHBCSMzUvAVYnLRChYkNOEl8RcYGRomJygpKjU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6goOEhYaHiImKkpOUlZaXmJmaoqOkpaanqKmqsrO0tba3uLm6wsPExcbHyMnK0tPU1dbX2Nna4uPk5ebn6Onq8vP09fb3+Pn6/9oADAMBAAIRAxEAPwDz25X54gMD5wTg8/lWgti90+xCoZl+UucAnsPqarLa71WUAZVhn3rodOsTLIGWEM2QcnnpWlKnzSM69ZQia+geAtHm0yK+vLu4n8wZMcShApBwQzHJ7dcYquvhjS9M1mS1Wza8WZllH2iQxpBb8+Y5cEcg4wTx7ZrsrW0ntmgtLSLYoUS72XBUntzVbVLQ3dlqbzSM14sRjYOjbRHkEYI4JyD14FUovmv9kw9suXf3uqueQ3sMC3k62rs9uJGETHqUycE/hiqvzDgBiPpWnd2sluSGjOPesuSQK5yprKpHlZ20Z8yNSC6RLRY2xubHTtg12OhX1js2sSJTgA43H2wM5/pXnLdVrZ0N25O45yBnNaUqjRhXopq51N74n8Y2moy2lvNMunuysWSMPsVcbtrYOM0/TvFPiu5vli1aZpNOiiZAWRYw2Tw7YHJxVCaeYeSBLJgu+RuPODxTPOkZ5w0jkccFjTVFfGZe1+xbpuSa29tO5IG7JwCjbjXFXUWJiFYEflUt27ieQBmADkAZ6VQlJMhJJzUVZX3OmhDl2P/Z
---

Here is a simple Hello world program written in Python. All good, given the sufficient permission, it should work fine as expected.

<h5 a><strong><code>hello_world.py</code></strong></h5>

```python
#! /usr/bin/python3

print("Hello World!!!")
```

```console
foo@bar:~$ chmod +x hello_world.py

foo@bar:~$ ./hello_world.py
Hello World!!!
```

And it does, what if I modify `hello_world.py` to

```python
#! /./././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././usr/bin/python3


print("Hello World!!!")
```

Hmmm.., the shebang's a bit longer. But if you scroll to the right, the specified path is same as that of the previous shebang `/usr/bin/python3`. So, the shebang's long, it shouldn't matter much right ? Let's give the required permission and try executing the file.

```console
foo@bar:~$ chmod +x hello_world.py

foo@bar:~$ ./hello_world.py
zsh: ./shebangtest.py: bad interpreter:  /././././././././././././././././: exec format error
```

It looks like the Kernel's got some problem if the shebang is too long. Why ? How long can the shebang be ? Let's see.

## Shebang

Before going forward, let's just refresh what a shebang is. A shebang is the first line in a script file that tells the operating system which interpreter to use to execute the script.

```python
#! /usr/bin/python3
```

Here we are telling the OS to use Python as the interpreter to execute this script.

## Why can't we have long shebangs

To understand the reason, let's get a rough idea of what happens when we execute the script using `./hello_world.py`.

When we execute a file, the kernel creates a `struct` with some data about the file. Like the file name, arguments, etc. One of the keys in this struct is `buf` which stores the first 256 bytes of the file to be executed.

After creating this `struct`, the kernel iterates through different binary format handlers which checks `buf` from the `struct` to identify whether the particular file is supported by them. Script file with shebangs are supported by a format handler called `binfmt_script.c`. It checks if the file begins with `#!` and parses the interpreter from the line.

Remember how `buf` contains only the first 256 bytes of the file. When we have a long shebang like the 2nd python script, the whole path isn't visible to `binfmt_script.c` and it tries to load the interpreter from the wrong path and ends up raising a "bad interpreter" error.

Till now we have been talking about the Linux kernel, where the size is defined as a constant `BINPRM_BUF_SIZE`. By trying out different sizes of shebangs, I found out that the limit is 512 in MacOS.

The main reason for limiting the length of buf is for efficiency. And almost all the time, the first 256 bytes is enough to determine the respective binary format handler.

## Conclusion

Long shebangs can cause issues because the kernel typically only reads the first 256 (or 512 in the case of macOS) bytes of the file to determine its format. This buffer size limitation ensures efficient handling of files without unnecessary overhead. When the shebang is too long, the kernel might not correctly identify the interpreter, leading to errors."

## Sources
- [cpu.land](cpu.land)
