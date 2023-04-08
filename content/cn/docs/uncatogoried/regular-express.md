---
title: "正则表达式"
linkTitle: "正则表达式"
date: 2023-04-08
description: >
 .
---

​
# 介绍

正则表达式介绍：正则表达式

# Python example

```python
#!/usr/bin/python3 -u

import re

# \d   [0-9]
p=re.compile('\d')
print(p.search('zy3t4r1').group())
print(p.findall('zy3t4r1'))

# \w [A-Za-z0-9]
q=re.compile('\w')
print(q.search('1a2b3c').group())
print(q.findall('1a2b3c'))

# \s space, \t\r\n space,
# \S not space
l=re.compile('\s')
print(l.search('\rab1 \ncd2 \tef3').group())
print(l.findall('\rab1 \ncd2 \tef3'))

# 
m=re.compile('[abc]?')
print(m.match('abcdefabc').group())
print(m.search('abcdefabd').group())
print(m.findall('abcdefabd'))

# {n}
n=re.compile('[dc]{2}')
print(n.search('ddattaccstle').group())
print(n.findall('ddattaccstle'))

# {n, m}
o=re.compile('[dc]{2,4}')
print(o.match('ddattacccstle').group())
print(o.search('ddattacccstle').group())
print(o.findall('ddattacccstle'))

# group
a=re.search(r'(\d{3})-(\d{3})-(\d{4})','My phone number is 022-888-5566')
if (a != None):
    print(a.group(1))
    print(a.group(2))
    print(a.group(3))
    print(a.group())

# $
b=re.compile('[abc]$')
print(b.findall('abcefAbc'))

# "\A"
c=re.compile('\A[a]+')
print(c.search('aabbccaaa').group())
print(c.findall('aabbccaaa'))

```
​
# Golang Example

```golang
package main

import (
	"bytes"
	"fmt"
	"regexp"
)

func directMatch() {
	match, err := regexp.Match(`a(.*)c`, []byte("abc"))
	fmt.Printf("%v, %v\n", match, err)

	match, err = regexp.MatchString(`a(.*)c`, "abc")
	fmt.Printf("%v, %v\n", match, err)

	runeReader := new(bytes.Buffer)
	runeReader.WriteString("abc")
	match, err = regexp.MatchReader(`a(.*)c`, runeReader)
	fmt.Printf("%v, %v\n", match, err)
}

func compileMatch() {
	compile, err := regexp.Compile(`a(.*)c`)
	if err != nil {
		panic("compile regexp failed")
	}
	fmt.Printf("%v\n", compile.MatchString("abc"))

	compilePOSIX, err := regexp.CompilePOSIX(`a(.*)c`)
	if err != nil {
		panic("compile regexp failed")
	}
	fmt.Printf("%v\n", compilePOSIX.MatchString("abc"))

	compile = regexp.MustCompile(`a(.*)c`)
	fmt.Printf("%v\n", compile.MatchString("abc"))

	compilePOSIX = regexp.MustCompilePOSIX(`a(.*)c`)
	fmt.Printf("%v\n", compilePOSIX.MatchString("abc"))
}

func compileFind() {
	compile, err := regexp.Compile(`a(b*)c`)
	if err != nil {
		panic("compile regexp failed")
	}
	fmt.Printf("%v\n", compile.Find([]byte("abcacabbbcddddabcddddac")))
	fmt.Printf("%v\n", compile.FindAll([]byte("abcacabbbcddddabcddddac"), -1))
	fmt.Printf("%v\n", compile.FindString("abcacabbbc"))
	fmt.Printf("%v\n", compile.FindAllString("abcacabbbcddddabcddddac", -1))

	fmt.Printf("%v\n", compile.FindSubmatch([]byte("abcacabbbcddddabcddddac")))
	fmt.Printf("%v\n", compile.FindStringSubmatch("abcacabbbcddddabcddddac"))

	fmt.Printf("%v\n", compile.FindAllSubmatch([]byte("abcacabbbcddddabcddddac"), -1))
	fmt.Printf("%v\n", compile.FindSubmatch([]byte("abcacabbbcddddabcddddac")))
	fmt.Printf("%v\n", compile.FindStringSubmatch("abcacabbbcddddabcddddac"))
	fmt.Printf("%v\n", compile.FindAllStringSubmatch("abcacabbbcddddabcddddac", -1))
}

func compileFindIndex() {
	compile, err := regexp.Compile(`a(b*)c`)
	if err != nil {
		panic("compile regexp failed")
	}
	fmt.Printf("%v\n", compile.FindIndex([]byte("abcacabbbcddddabcddddac")))
	fmt.Printf("%v\n", compile.FindAllIndex([]byte("abcacabbbcddddabcddddac"), -1))
	fmt.Printf("%v\n", compile.FindStringIndex("abcacabbbc"))
	fmt.Printf("%v\n", compile.FindAllStringIndex("abcacabbbcddddabcddddac", -1))

	fmt.Printf("%v\n", compile.FindSubmatchIndex([]byte("abcacabbbcddddabcddddac")))
	fmt.Printf("%v\n", compile.FindStringSubmatchIndex("abcacabbbcddddabcddddac"))
	fmt.Printf("%v\n", compile.FindAllSubmatchIndex([]byte("abcacabbbcddddabcddddac"), -1))
	fmt.Printf("%v\n", compile.FindAllStringSubmatchIndex("abcacabbbcddddabcddddac", -1))
}

func compileReplace() {
	compile, err := regexp.Compile(`a(b*)c`)
	if err != nil {
		panic("compile regexp failed")
	}
	fmt.Printf("%v\n", []byte("abcacabbbcddddabcddddac"))
	fmt.Printf("%v\n", compile.ReplaceAll([]byte("abcacabbbcddddabcddddac"), []byte("xyz")))
	fmt.Printf("%v\n", compile.ReplaceAll([]byte("abcacabbbcddddabcddddac"), []byte("$1")))

	fmt.Printf("%v\n", "abcacabbbcddddabcddddac")
	fmt.Printf("%v\n", compile.ReplaceAllString("abcacabbbcddddabcddddac", "xyz"))
	fmt.Printf("%v\n", compile.ReplaceAllString("abcacabbbcddddabcddddac", "$1"))

	fmt.Printf("%s\n", compile.ReplaceAllLiteral([]byte("...abc..."), []byte("xyz")))
	fmt.Printf("%s\n", compile.ReplaceAllLiteralString("...abc...", "xyz"))
	fmt.Printf("%s\n", compile.ReplaceAllFunc([]byte("...abc..."), func(bs []byte) []byte { return []byte("$1") }))
	fmt.Printf("%s\n", compile.ReplaceAllStringFunc("...abc...", func(s string) string { return "replace" }))
}

func main() {
	directMatch()
	compileMatch()
	compileFind()
	compileFindIndex()
	compileReplace()
}
```