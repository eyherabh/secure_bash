# Secure bash

Notes and tips beyond those found in [[1]] to make bash scripts safer, more resilient, and more portable. The notes and tips are based on actual scripts written by developers with various levels of maturity for operating on a range of systems which diverse and potentially outdated versions of bash are out of their control.


+ [Associative arrays within functions cannot be made global](#associative-arrays-within-functions-cannot-be-made-global)
+ [Assuming that `ls` starts with `.` and `..`](#assuming-that-ls-starts-with-.-and-..)
+ [Caution when mixing common and designated initializers](#caution-when-mixing-common-and-designated-initializers)
+ [Incomplete traversal of indexed arrays](#incomplete-traversal-of-indexed-arrays)
+ [Quoting and `printf %q` does not prevent word splitting](#quoting-and-printf-%q-does-not-prevent-word-splitting) 
+ [Strings in integer-valued variables](#strings-in-integer-valued-variables)
+ [Uninitialized array elements are left uninitialized](#uninitialized-array-elements-are-left-uninitialized)
+ [Validating integer values](#validating-integer-values)

## Associative arrays within functions cannot be made global

Consider the following script
```
a() { 
    declare -ag A=([0]=1 [1]=2) 
    printf "Within a: $(declare -p A)\n"
}

b() { 
    declare -Ag B=([0]=1 [1]=2) 
    printf "Within b: $(declare -p B)\n"
}

unset -v A B

a
printf "After a: $(declare -p A)\n"

b
printf "After b: $(declare -p B)\n"
```
which produces the following output
```
Within a: declare -a A='([0]=1 [1]=2)'
After a: declare -a A='([0]=1 [1]=2)'
Within b: declare -A B='()'
After b: declare -A B='()'
```
Turns out that the flag `-g` passed to `declare` causes associative arrays to be left empty. This is a known bug that has already been fixed, but it is still relevant, for example, in systems shipping bash 4.2 (e.g. CentOS 7).

Analogous effects are observed when attempting to use `-g`, for example, in the same way as `-r`, that is
```
c() {
    declare -a C=(1 2)
    declare -g C
}
```
This may be argued as documented and hence expected. Specifically, `help declare` states that the flag `-g` is used at the moment of creation as opposed to setting attributes. Nevertheless, current versions of bash seem to preserve the values, albeit not turning a variable previously declared as local into a global one.


## Assuming that `ls` starts with `.` and `..`

The list produced by `ls -a` need not start with `.` followed by `..`, as for example. Instead, I found that files named `^`, `<`, `=`, `>`, `_`, `:`, `;`, `!`, and `?` are all listed first. For example, the files named `_` and `?` will not appear as below

```bash
> ls -a
.
..
_
?
```
but like this instead
```bash
> ls -a
_
?
.
..
```
The list of file names appearing before `.` and `..` mentioned above is not exhaustive, and their order of appearance might well depend on the bash version and locale. 

Those file names may indeed be regarded as extremely unlikely, yet they may still occur by error or on purpose and break scripts. For example, a script relying on `.` and `..` appearing first may attempt to ignore them through `ls -1 | tail -n +3`. That script would have failed in the example above. Excluding them can be achieved through `ls -A`. 

It may sound contrived that one would use `ls -1 | tail -n +3` instead of `ls -A`, particularly since the latter is shorter. However, one may well have ignored the latter construct and missed it when reading the man page for `ls`.


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

## Quoting and `printf %q` does not prevent word splitting

Complex calls are sometimes made by first building the command as a concatenation of strings and then expanding the string, as for example
```
set "." 1 1 "f o o" "bar"
cmd="find"
cmd+=" ${1:-.}"
cmd+=" ${2:+-mindepth $2}"
cmd+=" ${3:+-mindepth $3}"
cmd+=" ${4:+-name $4}"
cmd+=" ${5:+-not -name $5}"
$cmd
```
Many problems exist with above, one of which is concerned with word-splitting. The above expands to
```
<find> <.> <-mindepth> <1> <-mindepth> <1> <-name> <foo> <-not> <-name> <bar>
```
where the angle brackets are used as delimiters. However, replacing `"foo"` with `"f o o"` would have resulted in
```
<find> <.> <-mindepth> <1> <-mindepth> <1> <-name> <f> <o> <o> <-not> <-name> <bar>
```
revealing that word-splitting has taken place for the fourth parameter. Typical attempts to prevent this consist in using single/double quotes or using `printf %q`. For example, using 
```
cmd+=" ${4:+-name '$4'}"
```
would result in `$cmd` expanding to
```
<find> <.> <-mindepth> <1> <-mindepth> <1> <-name> <'f> <o> <o'> <-not> <-name> <bar>
```
whereas using
```
cmd+=" ${4:+-name $(printf "%q" "$4")}"

```
would result in `$cmd` expanding to
```
<find> <.> <-mindepth> <1> <-mindepth> <1> <-name> <f\> <o\> <o> <-not> <-name> <bar>
```
Neither attempt has prevented word-splitting nor protect from injections, and both leave unintended characters after the parameter substitution. 

These issues can be solved in at least two ways: by not evaluating through parameter substitution (described in another section), and by using arrays. In the latter case, the complex call could be constructed as follows
```
set "." 1 1 "f o o" "bar"
cmd=("find")
cmd+=("${1:-.}")
cmd+=(${2:+-mindepth "$2"})
cmd+=(${3:+-mindepth "$3"})
cmd+=(${4:+-name "$4"})
cmd+=(${5:+-not -name "$5"})
"${cmd[@]}"
```
which expands to
```
<find> <.> <-mindepth> <1> <-mindepth> <1> <-name> <f o o> <-not> <-name> <bar>
```





## Strings in integer-valued variables

As detailed in [[2]] (section 3.4), when a variable has its integer attribute, any value assigned to it undergoes arithmetic expansion. For example, the following snippet
```
unset A B C D E F
E=1 F=2A
declare -i A="D" B="E" C="F"
``` 
will result in `A` set to `0` (zero), `B` set to `1` (unity), and `C` set but empty with the assignment producing the error
```
bash: declare: 2A: value too great for base (error token is "2A")
```
However, if the integer attribute is set for an existing variable, the arithmetic expansion does not occur, and most importantly, the variable remains with its value unchanged. As a result, the presence of the integer attribute guarantees neither that the value of the variable is a valid integer nor that expressions expecting integers will succeed.

For example, the following snippet
```
unset A B
declare A="B"
declare -i A
declare -p A
[ $A == B ] && printf yes || printf  no
[ $A -ge 0 ] && printf yes || printf  no

```
will produce 
```
declare -i A="B"
yes
bash: [: B: integer expression expected
```
Note that the integer attribute is set but the value of `A` is not an integer without any warning or error messages at the moment of setting the integer attribute on. String comparisons continue being successful whereas integer comparisons failed despite the integer attribute being set. 

Consequently, to be on the safe side, recall that

+ `declare` does not redeclare.
+ activated attributes are no guarantee of enforced attributes.


## Uninitialized array elements are left uninitialized

Unlike C [[3]], uninitialized array elements are left uninitialized, even if the integer attribute was set. That is, the following line
```
declare -ai A=(1 [2]=2)
```
does not initialize `A[1]` to zero as it would be in C [[3]]. The same applies to single variables. As a result, integer comparisons are not always guaranteed to succeed for variables even if their integer attribute is set.


## Validating integer values

Integer values can be validated as follows
```
case "$val" in [!+-0-9]*[!0-9]*) false;; *) [ $val -eq $val ] >&/dev/null;; esac
```

The first case is activated unless `val` starts with `+`, `-`, or a digit, and all subsequent characters are digits. Otherwise, the second case is activated, and the test returns true if the value is within range, and false if it is not.

## References

[1]: https://mywiki.wooledge.org/BashPitfalls
[2]: https://www.gnu.org/software/bash/manual/html_node/Arrays.html
[3]: http://www2.open-std.org/JTC1/SC22/WG14/www/abq/c17_updated_proposed_fdis.pdf

1. [Bash pitfalls](https://mywiki.wooledge.org/BashPitfalls)
2. [Bash reference manual](https://www.gnu.org/software/bash/manual/html_node/Arrays.html)
3. [C17 standard draft N2176](http://www2.open-std.org/JTC1/SC22/WG14/www/abq/c17_updated_proposed_fdis.pdf)
