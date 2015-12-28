---
layout: post
title:  "Markov models"
date:   2015-12-28 11:23:00
categories: markov model probability machine learning
---

A few days ago, a colleague told me about *hidden Markov models* while we were
speaking of autocomplete features in phones. Here is a screenshot of me writing
`How are` on my phone while it presents me some probable next words for this
sentence including the right one `you`.

![Autocomplete feature on my phone]({{ site.url }}/assets/markov-models/autocomplete.jpg)

Now, let me be clear: I am AI ignorant, I never took time to study/play with AI
and machine learning concepts and I feel shameful for this. One of the main
reason for this, I guess, is that I did not know where to start from.
Sometimes, people ask me where to start when you want to learn programming, and
I always tell them that they should try to create something, and learn specific
things as they need them, because no guide/tutorial/book is good enough.
For me, this experience-centered approach has always been proven right, and
guess what? It worked again    .

As the old saying goes "the best way to learn something is to teach it", I want
to share with you my modest knowledge about Markov models so we can (hopefully)
learn something together. That being said, I do not want to contradict what I
just wrote: "no guide/tutorial/book is good enough" and that is why this article
will be more of a log of my learning journey than a lesson or a whitepaper. It
may contain mistakes (and if you find some, please tell me), it might not be
complete, it might go too far, but hey **at least it reflects the reality of
learning**.

## First experiment

So my colleague told me that this could be done by using *hidden markov models*.
I asked him to tell me a bit more. He told me that by looking at large texts
(*corpus*), you could determine the probability for a word in the corpus to be
followed by another word, and then, when this word appear, predict using these
probabilities, the word that will follow.

Here is an example of corpus:

```
the computer of the son of the daughter of the sister of Mary
```

Here we can see that the word `of` is followed by `the` three times, and by
`Mary` once. Then, it is more likely when we encounter the word `of` that it
will be followed by `the` than by `Mary`. This can be intuitively described by
an array:

|              | Mary | son | of | sister | computer | daughter | the |
|--------------|-----:|----:|---:|-------:|---------:|---------:|----:|
| Mary         | 0    | 0   | 0  | 0      | 0        | 0        | 0   |
| son          | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| of           | 1    | 0   | 0  | 0      | 0        | 0        | 3   |
| sister       | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| computer     | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| daughter     | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| the          | 0    | 1   | 0  | 1      | 1        | 1        | 0   |

Here we can see that the word `of` has been followed three times by `the`, and
once by `Mary`. Here, we count the number of occurrences, but it might by more
useful to convert this to probabilities by dividing each number of occurrences
by the total number of occurrences (the number of words):

|              | Mary  | computer | of    | sister | daughter | the  | son   |
|--------------|------:|---------:|------:|-------:|---------:|-----:|------:|
| Mary         | 0     | 0        | 0     | 0      | 0        | 0    | 0     |
| computer     | 0     | 0        | 0.077 | 0      | 0        | 0    | 0     |
| of           | 0.077 | 0        | 0     | 0      | 0        | 0.23 | 0     |
| sister       | 0     | 0        | 0.077 | 0      | 0        | 0    | 0     |
| daughter     | 0     | 0        | 0.077 | 0      | 0        | 0    | 0     |
| the          | 0     | 0.077    | 0     | 0.077  | 0.077    | 0    | 0.077 |
| son          | 0     | 0        | 0.077 | 0      | 0        | 0    | 0     |

We can now quickly answer questions like "what is the probability for
two contiguous words in the whole text to be `of the`" (it is 0.23).
But it is not practical to answer queries like "what is the probability for the
word `the` to be next knowing that the current word is `of`" (this is what is
used in autocomplete).

It would be far easier for each line in the array to have the probability for
the word in the line to be followed by another word. So instead of dividi§ng each
number of occurrences by the number of words, we can divide it by the sum of
occurrences in the line (that is, the number of times the word appeared followed
by a word). For example, the word `of` has been followed by `Mary` once and
`the` three times. The total number of occurrences is then 4.
We can divide all the occurrences in the line `of` by 4, and do the same thing
for every line in the array:

|              | Mary | son | of | sister | computer | daughter | the |
|--------------|-----:|----:|---:|-------:|---------:|---------:|----:|
| Mary         | 0    | 0   | 0  | 0      | 0        | 0        | 0   |
| son          | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| of           | 0.25 | 0   | 0  | 0      | 0        | 0        | 0.75|
| sister       | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| computer     | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| daughter     | 0    | 0   | 1  | 0      | 0        | 0        | 0   |
| the          | 0    | 0.25| 0  | 0.25   | 0.25     | 0.25     | 0   |


This simple "array" now becomes a so-called
[*stochastic matrix*][stochastic-matrix]: it means that the sum of the
probabilities of a line is 1 (whereas in the previous example, it was the sum of
all probabilities in the matrix that equated 1).
Interesting fact: this matrix is also called *Transition matrix* (because it
reflects the probability for transiting from a word to another), or even
*Markov matrix*.

We this matrix, we can clearly see that the probability for a word to be `the`
knowing the previous word is `of` is 0.75. Guess what? With this simple query,
we can now code our own autocomplete feature.

[I am courageous enough to share with you the horrible     code I wrote in 10
minutes to test this][experiment-1-gist]. Do not try to run the code, you might
miss dependencies, but here's the result:

![Script output]({{ site.url }}/assets/markov-models/script-output.png)

We try train our Markov model with the sentence `the computer of the son of the
daughter of the sister of Mary`, print the occurrences matrix, and try to
predict the word following `of` (which is often `then` as shown, but could also
be `Mary`).

## Trying to generalize

Not only the previous code is ugly, but it is reaaaaaally slow and limited to
text, so I decided that I could try to write something to generalize the
"hidden Markov model" but [you can find another implementation here][github]
(we will talk about it later).

[stochastic-matrix]: https://en.wikipedia.org/wiki/Stochastic_matrix
[experiment-1-gist]: https://gist.github.com/brendan-rius/f60d7326aee4e33c5aaf
[github]: https://github.com/brendan-rius/markov-model
