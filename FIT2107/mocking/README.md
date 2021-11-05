# Mocking

## Reason for doubling or mocking?

* The real code you need to bypass has **unpredictable behaviour** - for instance, it interacts with an external system or users.
* The target for doubling doesn't exist yet
* The target for doubling is slow, making unit test run slowly. (Latency issue with server)
    * Digression: Disks (even SSDs) are slow, Networks are glacial
* The target for doubling interacts with software or hardware that doesn't exist in the unit testing environment.
* You want to test your test target with cases that are rarely if ever generated by the target for doubling, such as error cases.
* You want to **inspect how the code you're actually testing calls the target for doubling**, and the target doesn't provide facilities for doing this.

```python

class​ CharCounter:
​'''Class for counting the number of instances of characters in a
file'''
​def​ ​__init__​(self, infile):
  ​"""Construct the class
  Arguments:
  infile: the file object to count
  """
  self.counts = ​{}
  self.buildcounts(infile)

​def​ ​buildcounts​(self, infile):
  ​"""Build the count for a file. This is called by the constructor
  so should not typically be called directly
  """
  chars = infile.read()
  ​for​ char ​in​ chars:
  ​if​ char ​in​ self.counts:
  self.counts[char] += ​1
  ​else​:
  self.counts[char] = ​1

​def​ ​getcount​(self, letter):
  ​"""Look up the count for a particular character
  Arguments:
  letter: the character to get the count for
  Returns: the number of times the letter occurs, which may be
  zero if it is not present
  """
  ​if​ letter ​in​ self.counts:
  ​return​ self.counts[letter]
  ​else​:
  ​return​ 0

``**

## Fakefile

**Example:**
To test the ​CharCounter​ class, we need an object that implements a **read method** that **returns a list (or other iterable) of single-character strings**.

We can use Fakefile as a **stub** (test double) to test CharCounter without actually accessing files:

```python
class​ ​FakeFile​:
  ​def​ ​__init__​(self, contents):
    self.contents = list(contents)

  ​def​ ​read​(self):
    return self.contents
```

```python
from​ charcounter ​import​ CharCounter
from​ fakefile ​import​ FakeFile
import​ unittest
class​ ​CharCounterHandTest​(unittest.TestCase):
  ​def​ ​test_one​(self):
    infile = FakeFile(​"fredfredfred"​)
    counter = CharCounter(infile)

    self.assertEqual(counter.getcount(​"f"​),​ 3​)

    if​__name__==​"__main__"​:
      unittest.main()
```

## Problems

### Problem 1: Intercepting indirect function/method invocations

```python
class​ CharCounter:
  ​ '''Class for counting the number of instances of characters in a
  file'''
  ​def​ ​__init__​(self, inpath):
    ​"""Construct the class
    Arguments:
    inpath: a path to the fil to count
    """
    ​self.counts = ​{}
    self.buildcounts(inpath)

  ​def​ ​buildcounts​(self, inpath):
    """Build the count for a file. This is called by the constructor
    so should not typically be called directly
    """
    infile = open(inpath, ​"r"​)
  ​for​ char ​in​ chars:
  ​if​ char ​in​ self.counts:
  self.counts[char] += ​1
  ​else​:
  self.counts[char] = ​1
  # Rest of the code is unchanged....

```

We can't pass in a ​FakeFile​ object now. 

#### Alternate version of ​CharCounter

Without accessing files, we need to somehow trap calls to ​open()​ and get it to return a test double for ​infile​.

In a statically typed language like Java, replacing the real ​open()​ with a fake ​open()​ would be extremely difficult. It's actually not as hard in Python, as this example (non-test) code demonstrates:

* You have to be very disciplined about the scope of your substitution, as there could be all manner of bugs. This is quite complicated in unit testing, as the test runners may execute your tests in ​any​ order. 

* The code for substituting one function for another was relatively simple. **Substituting an instance method attached to a class results in considerably more complex code**.


```python
from​ fakefile ​import​ FakeFile

def​ ​myopen​(fakefile, fakemode):
  ​return​ FakeFile(​"this is a test"​)

properopen = open

#substitute in the fake open function
open = myopen

#calls the fake open function and prints "this is a test"
myfake = open(​"/proc/version_signature"​,​"r"​)
print(myfake.read()) # Printing the stubbed version of reading and opening a file. It should print This is a test

#switch the real open back in
open = properopen
myreal = open(​"/proc/version_signature"​, "r")
print(myreal.read())


```

### Problem 2: Monitoring the way functions are invoked

The point of unit testing is to check whether the software under test actually behaves as expected. Those expectations can include that it invokes other code correctly.

To take a very simple example, we might want to check that ​CharCounter​ is opening the file we ask it to count, and that the read mode is "r".

Again, it is ​possible​ to handcraft test doubles to do this (if you want to learn about some fairly advanced aspects of Python, it's a good exercise to attempt to do it cleanly), but the code to do so gets increasingly complex as the checking becomes more elaborate.
