+++
title = "The even-ended number problem in Go and Python"
date = "2020-10-20T10:44:00+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Golang","Development","Performance","Python",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2020/10/the-even-ended-number-problem-in-go-and.html']
+++
During the Go Essential Training course on LinkedIn, the instructor sets up a problem for you to solve. The solution is in the next slide of the course, and mine was ever so slightly different anyways, so I doubt that this needs to come with a spoiler alert, but anyways...

An even-ended number is one that starts and ends with the same digit. That is to say 1, 11, 121, and 10103921 are all even ended numbers as they all start and end with the same digit.

The problem posed, is how many even-ended numbers can be found in the multiplication of all combinations of two 4 digit numbers.

In other words, if you multiple all the numbers in the range 1000 - 9999, by all the numbers in the range 1000-9999, how many of the results are even ended numbers.

The Go solution I came up with first generated about 6million. This was when I started to wonder if I was double counting, since 1001x1011 and 1011x1001 are the even-ended, but also the same result, meaning I counted 2 when I only needed to count 1. Rather than just cheat and divide by 2, I instead made the second loop seed the number from the first loop to ensure that we only count pairs once.

Here is the Go Solution I wrote:

```golang
package main

import (
        "fmt"
)

// how many even ended numbers result from multiplying two four digit numbers

func main() {
        // setup a new counter
        even_numbers := 0

        // setup a loop with x as a seed
        for x := 1000; x <=9999; x++ {

                //iterate over that loop with y incrementing as well
                for y := x; y <= 9999; y++ {

                        // work out the answer
                        result := x*y
                        result_str := fmt.Sprintf("%d", result)

                        // check if the answer has the same start and end digit
                        if result_str[0] == result_str[len(result_str)-1] {
                                // increment the counter
                                even_numbers++
                        }
                }
        }
        // at the end, print out the count...
        fmt.Printf("We found %d even-ended numbers", even_numbers)
        fmt.Println("")

}
```

Checking that with the solution in the course, its pretty much identical, and the result was the same. I was then telling a friend how much quicker it was without all the debug fmt.println() crap I had used to validate each stage of the loop. His response was that golang is mad fast and to compare it to Python3.

Challenge accepted. Given I knew the logic I needed already I sort of shocked myself that I was able to port this to Python3 in the time it took me to retype it essentially. Save for one Typo, it ran first time.

Here is the Python3 Solution I wrote:

```python
#!/usr/bin/env python3

even_numbers = 0

x = 1000
y = 1000
 
while (x <= 9999):
    y = x
    while (y <= 9999):
        result = x * y
        if str(result)[0] == str(result)[len(str(result))-1]:
            even_numbers = even_numbers + 1
        y = y + 1
    x = x + 1
print(even_numbers)
```

What was a real eye opener for me, was the runtime stats.

```shell
% time go run even-numbers.go
We found 3184963 even-ended numbers
go run even-numbers.go  6.09s user 0.50s system 106% cpu 6.202 total

% time python3 even_numbers.py
3184963
python3 even_numbers.py  59.78s user 0.20s system 99% cpu 1:00.03 total
```

Literally 10x the speed difference between python3 and Golang. Bonkers.

I can see why people love Go as an alternative to python. Both are completely portable between operating systems and architectures (caveat emptor not withstanding), and both are very readable, and approachable. Go is just MUCH faster.

I'm enjoying the learning experience so far and apart from endlessly using pythonic syntax within the go space almost every single day, I'm making a lot of progress.

I still expect my first "real" golang project will be an API service replacing something old and crusty I already have in python. When I am ready to go there (geddit!), I will be sure to write it up here of course.
