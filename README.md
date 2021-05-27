# Secure bash

Notes and tips beyond those found in [[1]] to make bash scripts safer, more resilient, and more portable in the face of real-world production application having to perform in multiple possibly outdated systems. 


+ [Associative arrays within functions cannot be made global](#associative-arrays-within-functions-cannot-be-made-global)
+ [Assuming that `ls` starts with `.` and `..`](#assuming-that-ls-starts-with-.-and-..)
+ [Caution when mixing common and designated initializers](#caution-when-mixing-common-and-designated-initializers)
+ [Incomplete traversal of indexed arrays](#incomplete-traversal-of-indexed-arrays)
+ [Quoting and `printf %q` does not prevent word splitting](#quoting-and-printf-%q-does-not-prevent-word-splitting) 
+ [Strings in integer-valued variables](#strings-in-integer-valued-variables)
+ [Uninitialized array elements are left uninitialized](#uninitialized-array-elements-are-left-uninitialized)
+ [Uninitialized variables undetected by `set -u`](#uninitialized-variables-undetected-by-set--u)
+ [Validating integer values](#validating-integer-values)

## Associative arrays within functions cannot be made global

In Bash 4.2, using `declare -g` within a function causes associative arrays to be left empty both inside and outside the function. This is a known bug and has already been fixed, but it is still relevant, for example, in systems like CentOS 7. To illustrate the issue, consider first the following script

```
a() { 
    declare -ag A=([0]=1 [1]=2) 
    printf "Within a: $(declare -p A)\n"
}
unset -v A 

a
printf "After a: $(declare -p A)\n"
```
which produces the following output
```
Within a: declare -a A='([0]=1 [1]=2)'
After a: declare -a A='([0]=1 [1]=2)'
```
In this case, the script created a global indexed array and, as expected, the values persist after `declare` both inside and outside the function. Now consider an analogous script that declares an associative array instead
```
b() { 
    declare -Ag B=([0]=1 [1]=2) 
    printf "Within b: $(declare -p B)\n"
}
unset -v B

b
printf "After b: $(declare -p B)\n"
```
which produces the following output
```
Within b: declare -A B='()'
After b: declare -A B='()'
```
Unlike the previous script, the output of this one shows that the created array is empty.

Analogous effects are observed when first declaring the array and then attempting to use `-g` as if it were setting an attribute (analogously to the `-r` flag)
```
c() {
    declare -a C=(1 2)
    declare -g C
}
```
except that this time it happens regardless of the array type. This observation may be argued as documented and hence expected. Specifically, `help declare` states that the flag `-g` is used at the moment of creation as opposed to setting attributes. Nevertheless, current versions of bash seem to preserve the values, albeit not turning a variable previously declared as local into a global one.


## Assuming that `ls` starts with `.` and `..`

The list produced by `ls -a` need not always start with `.` followed by `..`. Instead, I found that files named `^`, `<`, `=`, `>`, `_`, `:`, `;`, `!`, and `?` are all listed before them. Specifically, in a folder with files named `_` and `?`, `ls -a` will list them not as 
```bash
.
..
_
?
```
but as follows
```bash
_
?
.
..
```
The list of file names appearing before `.` and `..` mentioned above is not exhaustive, and their order of appearance might well depend on the bash version and locale. 

Those file names may indeed be regarded as extremely unlikely, yet they may still occur (by error or on purpose) and break scripts. Scripts relying on `.` and `..` appearing first and using `ls -1 | tail -n +3` to filter them are not that rare, and would have failed in the example above. Instead, `.` and `..` can be excluded by using `ls -A`, which is also shorter, or use `find`. 

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
```
declare -a A=([1]=1) && echo ${#A[@]}
```
will produce `1` instead of `2`. The same will occur after unsetting some of the array values, e.g.
```
declare -a A=(0 1) && unset "A[0]" && echo ${#A[@]}
```
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

## Uninitialized variables undetected by `set -u`

According to [2], one way to detect the use of unset (a.k.a. unbound) variables is through the shell option `nounset`, activated throuhg `set -u` or `set -o nounset`). After that, the shell will treat as an error any use of unset variables other than `@` and `\*` as an error. However, turns out that this is not always the case for array expansions (at least in Bash 5.0.17). 

Specifically, consider the following script
```
declare -a A
: ${A[0]}
```
In agreement with [2], the line immediately after the declaration is treated as an error. Indeed, it's trying to access an element that is unset. Analogous results are observed when running the following script
```
declare -a A=(1 2)
: ${A[2]}
```
This script differs from the previous one in that the array is initialized, but the array access is out of bounds. However, seemingly in contradiction with [2], the following script runs without error
```
declare -a A
: ${A[@]}
: ${A[*]}
: ${!A[@]}
: ${!A[*]}
```
The same result is observed when the array is declared as associative, namely `declare -A A`. In conclusion, the shell option `nounset` does not entirely prevent the use of unset variables in parameter expansion.

One way to solve this issue is by testing during parameter expansions through the construct `${par?msg}`. For example, the following script
```
declare -a A
: ${A[@]?}
: ${A[*]?}
: ${!A[@]?}
: ${!A[*]?}
```
treats all expansions as errors and produce some standard error message. Unfortunately, this strategy also has its problems as well. 

Consider the following script
```
declare -a A=(0 1 2)
echo "${A[@]}"
echo "${!A[@]}"
```
which, unsurprisingly, produces
```
0 1 2
0 1 2
```
However, consider now the same script but using the `${par?msg}` construct
```
declare -a A=(0 1 2)
echo "${A[@]?}"
echo "${!A[@]?}"
```
This time, the result is actually surprising
```
0 1 2
bash: 0 1 2: invalid variable name
```
Instead of simply printing the array indexes, the second line was treated as an error.


Even when the result is satisfactory, the potential benefits of using of the `nounset` option or `${par?msg}` (but see [4]) are limited by the fact that the standard error messages may be slightly confusing or misleading. For example, when `A` is unset, all `: ${A?}`, `: ${!A?}`, `: ${A[@]?}` and `: ${!A[@]?}` produce the message `parameter not set`. However, this is not the case when `A` is set to an unset variable `B`. In that case, `${A?}` and `: ${A[@]?}` produce no errors as expected, but `: ${!A?}` produces `invalid indirect expansion` whereas `: ${!A[@]?}` produces `parameter not set`.


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
[4]: https://mywiki.wooledge.org/BashFAQ/112

1. [Bash pitfalls](https://mywiki.wooledge.org/BashPitfalls)
2. [Bash reference manual](https://www.gnu.org/software/bash/manual/html_node/Arrays.html)
3. [C17 standard draft N2176](http://www2.open-std.org/JTC1/SC22/WG14/www/abq/c17_updated_proposed_fdis.pdf)
4. [What are the advantages and disadvantages of using set -u (or set -o nounset)?](https://mywiki.wooledge.org/BashFAQ/112)
