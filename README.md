# Secure bash

Notes and tips beyond those found in [[1]] to make bash scripts safer and more portabble.
## [Strings in integer-valued variables](#strings-in-integer-valued-variables)
## [Caution when mixing common and designated initializers](#caution-when-mixing-common-and-designated-initializers)
## [Uninitialized array elements are left uninitialized](#uninitialized-array-elements-are-left-uninitialized)
## [Incomplete traversal of indexed arrays](#incomplete-traversal-of-indexed-arrays)
## [Assuming that `ls -a` starts with `.` and `..`](#assuming-that-ls-starts-with-.-and-..)


## Strings in integer-valued variables

As detailed in [[2]] (section 3.4), when a variable has its integer attribute, any value assigned to it undergoes arithmetic expansion. For example, the following snippet
```
unset A B
declare -i A="B"
``` 
will result in `A` being assigned `0` (zero). However, the integer attribute can be set on an existing variable without any change in the value assigned to it. 

Specifically, consider the following snippet
```
unset A B
declare A="B"
declare -i A
declare -p A
```
will produce 
```
declare -i A="B"
```
Note that the integer attribute is set on but the value of `A` is not an integer but a string and no warning has been generated or conversions carried out. 

Consequently, to be on the safe side, remember that

+ `declare` does not redeclare.
+ Activated attributes are no guarantee of enforced attributes.


## Caution when mixing common and designated initializers

As mentioned in [[2]], and analogously to [[3]], the index of the current element to be initialized is given in the initializer (a.k.a. designated initializers). If none is given, then it is equal to the index of the previously initialized element plus unity. If none were initialized, then it is zero. For example, the following line declares the indexed array `A` with four initializers
```
declare -a A=(A [1]=B C [3]=D)
```
The first and third initializers are common, whereas the second and fourth are designated. The line is equivalent to 
```
declare -a A='([0]="A" [1]="B" [2]="C" [3]="D")'
```
(to test the equivalence, use `declare -p A`).

However, this feature seems prone to errors and makes them hard to trace. For example, the following line
```
declare -a A=(A [2]=B [1]=C D)
```
and
```
declare -a A=(A [2]=B [1]=C [2]=D)
```
are equivalent to 
```
declare -a A='([0]="A" [1]="C" [2]="D")
```
That is, in both cases, the value of `A[2]` was first assigned `B` and then silently overwritten with `D`. This behaviour is analogous to that for array initialization in C (see [[3]] 6.7.9_17-18).

Associative arrays require the index to be provided, thus rendering any multiple assignment more visible, but they still provide no way to avoid them or to be inform about them at run-time.


## Uninitialized array elements are left uninitialized

Unlike C [[3]], uninitialized array elements are left uninitialized, even if the integer attribute was set. That is, the following line
```
declare -ai A=(1 [2]=2)
```
does not initialize `A[1]` to zero as it would be in C [[3]]. The same applies to single variables. As a result, integer comparisons are not always guaranteed to succeed for variables even if their integer attribute is set.


## Incomplete traversal of indexed arrays

The length of an indexed array need not be equal to the index of the last element plus unity. For example, 
```declare -a A=([1]=1) && echo ${#A[@]}```
will produce `1` instead of `2`. The same will occur after unsetting some of the array values, e.g.
```declare -a A=(0 1) && unset "A[0]" && echo ${#A[@]}```
Hence, the following construction will not traverse over all set elements of `A`
```
for i in $(seq 1 ${#A[@]}); do 
    # Do something with A[i-1] ...
done
```
but, as detailed in [[2]], the following construction will certainly do
```
for i in ${!A[@]}; do 
    # Do something with A[i-1] ...
done
```

## Assuming that `ls` starts with `.` and `..`

The list produced by `ls -a` need not start with `.` followed by `..`. Instead, I found that files starting with `^`, `<`, `=`, `>`, `_`, `:`, `;`, `!`, and `?` are all listed before. The previous list is not exhaustive, and the listing order might well depend on the bash version and locale. Scripts simply excluding `.` and `..` can achieve this by using `ls -A`.

## References

[1]: https://mywiki.wooledge.org/BashPitfalls
[2]: https://www.gnu.org/software/bash/manual/html_node/Arrays.html
[3]: http://www2.open-std.org/JTC1/SC22/WG14/www/abq/c17_updated_proposed_fdis.pdf

1. [Bash pitfalls](https://mywiki.wooledge.org/BashPitfalls)
2. [Bash reference manual](https://www.gnu.org/software/bash/manual/html_node/Arrays.html)
3. [C17 standard draft N2176](http://www2.open-std.org/JTC1/SC22/WG14/www/abq/c17_updated_proposed_fdis.pdf)
