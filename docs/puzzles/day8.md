import Solver from "../../../../website/src/components/Solver.js"

# Day 8: Seven Segment Search

## Puzzle description

https://adventofcode.com/2021/day/8

## Solution of Part 1

### Modelling the Domain

First we will model our problem and parse the input into our model.

#### `Segment`

As there are a fixed range of possible display _segments_:
`a`, `b`, `c`, `d`, `e`, `f`, `g`, we will model them with an enumeration.

> An enumeration is used to define a type consisting of a set of named values.
Read [the official documentation](https://docs.scala-lang.org/scala3/reference/enums/enums.html)
for more details.

Our enumeration `Segment` looks like the following:

```scala
enum Segment:
  case A, B, C, D, E, F, G
```

#### `Digit`

Next, we will model the possible _digits_ of a display. Again, there are a fixed number of them
so we will use an enumeration.

A _digit_ is made by lighting a number of _segments_, so we will associate each digit with
the segments required to light it, we can do this by adding a parameter to `Digit`:

```scala
enum Digit(val segments: Segment*):
  case Zero extends Digit(A, B, C, E, F, G)
  case One extends Digit(C, F)
  case Two extends Digit(A, C, D, E, G)
  case Three extends Digit(A, C, D, F, G)
  case Four extends Digit(B, C, D, F)
  case Five extends Digit(A, B, D, F, G)
  case Six extends Digit(A, B, D, E, F, G)
  case Seven extends Digit(A, C, F)
  case Eight extends Digit(A, B, C, D, E, F, G)
  case Nine extends Digit(A, B, C, D, F, G)
```

In the above `case One extends Digit(C, F)` defines a `Digit`: `One` which has segments `C` and `F`.
Each `Digit` is defined with the correct configuration of segments.

#### Segment Strings, aka `Segments`

In the input we see many sequences of strings, such as `"fdcagb"`, which we will call
_segment strings_. We can think of a segment string as a set of characters
that represent the segments lit in a single digit of a display.
These segments are not necessarily the correct configuration for viewing by humans.

We will model a segment string in Scala by values of type `Set[Segment]`, which we will
simplify with a type alias `Segments`, defined in the companion object of
`Segment`:

```scala
object Segment:
  type Segments = Set[Segment]
```

### Finding A Unique `Digit` from `Segments`

The problem asks us to find segment strings that correspond to the digits with a unique number of segments.

First, let's find the digits that have a unique number of segments.

If we group the digits by the number of segments they have, we can see the following picture:

| No. Segments | Digits                       |
|--------------|------------------------------|
| 2            | { `One` }                    |
| 3            | { `Seven` }                  |
| 4            | { `Four` }                   |
| 5            | { `Two` , `Three` , `Five` } |
| 6            | { `Zero` , `Six` , `Nine` }  |
| 7            | { `Eight` }                  |

We can build the table above with the following code:
```scala
val bySizeLookup: Map[Int, Seq[Digit]] =
  index.groupBy(_.segments.size) // `index` is `Digit.values.toIndexedSeq`
```

> In the above, we can access all `Digit` values with the built in `values` method of the companion,
here we have converted it to a `Seq` (the `index` value) for convenience.

However we are only interested in the entries where the segment count is linked
to a single digit. We can remove the entries with more than one digit, and map each key to a single digit
with a `collect` call:
```scala
val uniqueLookup: Map[Int, Digit] =
  bySizeLookup.collect { case k -> Seq(d) => k -> d }
```

The content of `uniqueLookup` looks like the following:
```scala
Map(2 -> One, 3 -> Seven, 4 -> Four, 7 -> Eight)
```

For our problem, we need to lookup a `Digit` from a segment string, which we will implement
in the companion of `Digit`, with a method `lookupUnique`. The method takes a segement string
(of type `Segments`), and returns an `Option[Digit]`.

To implement this, we call `get` on
`uniqueLookup` with the size of the segment string, which returns an `Option[Digit]`, depending
on if the key was present.

Here is the final companion of `Digit` (we have removed `bySizeLookup` as it is not needed):

```scala
object Digit:

  val index: IndexedSeq[Digit] = values.toIndexedSeq

  private val uniqueLookup: Map[Int, Digit] =
    index.groupBy(_.segments.size).collect { case k -> Seq(d) => k -> d }

  def lookupUnique(segments: Segments): Option[Digit] =
    uniqueLookup.get(segments.size)
```

### Parsing the input

#### Parsing `Segments`

We will parse each segment string into `Segments` aka a `Set[Segment]`.

First, we add a `char` field to our `Segment` enum: its computed by making the first character of `toString` lower-case:

```scala
enum Segment:
  case A, B, C, D, E, F, G

  val char = toString.head.toLower
```

Next within `Segment`'s companion object we define:
- a reverse map `fromChar`, which maps each `Segment` by
  using its `char` field as the key.
- a method `parseSegments` which takes a segment string (typed as `String`)
  and converts it to a `Set[Segment]`, by looking up the corresponding
  `Segment` of each `Char` of the string in `fromChar`.

The final companion object to `Segment` can be seen below:

```scala
object Segment:
  type Segments = Set[Segment]

  val fromChar: Map[Char, Segment] = values.map(s => s.char -> s).toMap

  def parseSegments(s: String): Segments =
    s.map(fromChar).toSet
```

:::info
Please note that in `parseSegments` I am assuming that `fromChar` will contain each character of the input
string `s`, which is only safe with correct input. With invalid input it will throw `NoSuchElementException`.

To be more explicit with error handling, we could wrap `parseSegments` with a [Try](https://www.scala-lang.org/api/current/scala/util/Try.html).
:::

#### Parsing the input file

for part 1 we only need to read the four digit display section of each line, we can parse this from
a line with the following function:
```scala
def getDisplay(line: String): String = line.split('|')(1).trim
```

So far we have just a string, e.g. `"fdgacbe cefdb cefbgd gcbe"`. Next we extract each segment string
from a display by splitting on `' '`.

### Computing the Solution

Finally we want to lookup a possible unique digit for each segment string.

We will create a helper function `parseUniqueDigit` which first parses a segment string as
`Segments` and then looks up a unique `Digit`:

```scala
def parseUniqueDigit(s: String): Option[Digit] =
  Digit.lookupUnique(Segment.parseSegments(s))
```

Then, we will count the total unique digits found, we can see the computation of the solution
here:

```scala
// `input` is the input file as a String
val uniqueDigits: Iterator[Digit] =
  for
    display <- input.linesIterator.map(getDisplay)
    segments <- display.split(" ")
    uniqueDigit <- parseUniqueDigit(segments)
  yield
    uniqueDigit

uniqueDigits.size
```

### Final Code

The final code for part one is as follows:

```scala
import Segment.*

enum Segment:
  case A, B, C, D, E, F, G

  val char = toString.head.toLower

object Segment:
  type Segments = Set[Segment]

  val fromChar: Map[Char, Segment] = values.map(s => s.char -> s).toMap

  def parseSegments(s: String): Segments =
    s.map(fromChar).toSet

end Segment

enum Digit(val segments: Segment*):
  case Zero extends Digit(A, B, C, E, F, G)
  case One extends Digit(C, F)
  case Two extends Digit(A, C, D, E, G)
  case Three extends Digit(A, C, D, F, G)
  case Four extends Digit(B, C, D, F)
  case Five extends Digit(A, B, D, F, G)
  case Six extends Digit(A, B, D, E, F, G)
  case Seven extends Digit(A, C, F)
  case Eight extends Digit(A, B, C, D, E, F, G)
  case Nine extends Digit(A, B, C, D, F, G)

object Digit:

  val index: IndexedSeq[Digit] = values.toIndexedSeq

  private val uniqueLookup: Map[Int, Digit] =
    index.groupBy(_.segments.size).collect { case k -> Seq(d) => k -> d }

  def lookupUnique(segments: Segments): Option[Digit] =
    uniqueLookup.get(segments.size)

end Digit

def part1(input: String): Int =

  def getDisplay(line: String): String = line.split('|')(1).trim

  def parseUniqueDigit(s: String): Option[Digit] =
    Digit.lookupUnique(Segment.parseSegments(s))

  val uniqueDigits: Iterator[Digit] =
    for
      display <- input.linesIterator.map(getDisplay)
      segments <- display.split(" ")
      uniqueDigit <- parseUniqueDigit(segments)
    yield
      uniqueDigit

  uniqueDigits.size
end part1
```

<Solver puzzle="day8-part1"/>

## Solution of Part 2

### Modelling the Domain

For part 2 we can reuse all data structures from part 1.

### Cryptography and Decoding

In part 2 we are asked to decode each four digit display to an integer number. The problem is that each display
is wired incorrectly, so while the signals to the display are for a correct digit, the observable output is different.

We know that the configuration of wires is stable, so for each display we are given a list of the 10 possible visible
output signals of the display (one for each digit).

This problem is identical to a substitution cipher, where the alphabet is segments, and words are
digits. We will discover the alphabet used to encode the segments, and use it to decode encoded segment strings
into valid digits.

We will take the list of 10 unique output signals (words in the encoded alphabet) and produce
a dictionary of these words associated to valid words in the decoded alphabet
(aka a map of type `Map[Segments, Digits]`).

#### Discovering the Encoded Digits

In part 1 we discovered how to decode `One`, `Four`, `Seven` and `Eight` from a segment string because those
digits are made of a unique number of segments. Let's consider those as discovered:

TODO: make this a list of "rules", aka `one is the encoded string with the same number of segments as One`
```scala
// `uniques` is a Map from `Digit` to encoded digits with a unique number of elements.
val one = uniques(One)
val four = uniques(Four)
val seven = uniques(Seven)
val eight = uniques(Eight)
```

We are left with six more digits to discover:
- those with 5 segments: { `Two` , `Three` , `Five` }
- those with 6 segments: { `Zero` , `Six` , `Nine` }

We will group the 10 encoded digits by size, selecting those with 5 or 6 segments:

```scala
// cipher is our list of 10 encoded digits
val ofSizeFive = cipher.filter(_.sizeIs == 5)
val ofSizeSix = cipher.filter(_.sizeIs == 6)
```

Lets look again at the configuration of a valid display and its digits:
```
  0:      1:      2:      3:      4:
 aaaa    ....    aaaa    aaaa    ....
b    c  .    c  .    c  .    c  b    c
b    c  .    c  .    c  .    c  b    c
 ....    ....    dddd    dddd    dddd
e    f  .    f  e    .  .    f  .    f
e    f  .    f  e    .  .    f  .    f
 gggg    ....    gggg    gggg    ....

  5:      6:      7:      8:      9:
 aaaa    aaaa    aaaa    aaaa    aaaa
b    .  b    .  .    c  b    c  b    c
b    .  b    .  .    c  b    c  b    c
 dddd    dddd    ....    dddd    dddd
.    f  e    f  .    f  e    f  .    f
.    f  e    f  .    f  e    f  .    f
 gggg    gggg    ....    gggg    gggg
```

You might notice that some of the digits fit within another. For example, out of the 5 segment digits,
only `Three` has all the segments of `One`.

To find the segment string corresponding to `Three`, we will use a function `lookup`: it
takes a `Seq[Segments]` and a set of segments that uniquely identifies an encoded digit, returning
that encoded digit and the remaining `Seq[Segments]`.

```scala
val (three, remainingFives) = lookup(ofSizeFive, withSegments = one)
```


<!-- From our 10 encoded digits we can group the ones that have 5 or 6 segments:
```scala
// cipher is our list of 10 encoded `Segments`
val ofSizeFive = cipher.filter(_.sizeIs == 5)
val ofSizeSix = cipher.filter(_.sizeIs == 6)
``` -->

### Final Code

The final code for part 2 can be appended to the code of part 1:

```scala
import Digit.*

def part2(input: String): Int =

  def parseSegmentsSeq(segments: String): Seq[Segments] =
    segments.trim.split(" ").toSeq.map(Segment.parseSegments)

  def splitParts(line: String): (Seq[Segments], Seq[Segments]) =
    val Array(cipher, plaintext) = line.split('|').map(parseSegmentsSeq)
    (cipher, plaintext)

  def digitsToInt(digits: Seq[Digit]): Int =
    digits.foldLeft(0)((acc, d) => acc * 10 + d.ordinal)

  val problems = input.linesIterator.map(splitParts)

  val solutions = problems.map((cipher, plaintext) =>
    plaintext.map(substitutions(cipher))
  )

  solutions.map(digitsToInt).sum

end part2

def substitutions(cipher: Seq[Segments]): Map[Segments, Digit] =

  def lookup(section: Seq[Segments], withSegments: Segments): (Segments, Seq[Segments]) =
    val (Seq(uniqueMatch), remaining) = section.partition(withSegments.subsetOf)
    (uniqueMatch, remaining)

  val uniques: Map[Digit, Segments] =
    Map.from(
      for
        segments <- cipher
        digit <- Digit.lookupUnique(segments)
      yield
        digit -> segments
    )

  val ofSizeFive = cipher.filter(_.sizeIs == 5)
  val ofSizeSix = cipher.filter(_.sizeIs == 6)

  val one = uniques(One)
  val four = uniques(Four)
  val seven = uniques(Seven)
  val eight = uniques(Eight)
  val (three, remainingFives) = lookup(ofSizeFive, withSegments = one)
  val (nine, remainingSixes) = lookup(ofSizeSix, withSegments = three)
  val (zero, Seq(six)) = lookup(remainingSixes, withSegments = seven)
  val (five, Seq(two)) = lookup(remainingFives, withSegments = four &~ one)

  val decode: Map[Segments, Digit] =
    Seq(zero, one, two, three, four, five, six, seven, eight, nine)
      .zip(Digit.index)
      .toMap

  decode
end substitutions
```
<Solver puzzle="day8-part2"/>

## Run it locally

You can get this solution locally by cloning the [scalacenter/scala-advent-of-code](https://github.com/scalacenter/scala-advent-of-code) repository.
```
$ git clone https://github.com/scalacenter/scala-advent-of-code
$ cd advent-of-code
```

You can run it with [scala-cli](https://scala-cli.virtuslab.org/).

```
$ scala-cli src -M day8.part1
The solution is 521

$ scala-cli src -M day8.part2
The solution is 1016804
```

You can replace the content of the `input/day8` file with your own input from [adventofcode.com](https://adventofcode.com/2021/day/8) to get your own solution.

## Solutions from the community

Share your solution to the Scala community by editing this page.
