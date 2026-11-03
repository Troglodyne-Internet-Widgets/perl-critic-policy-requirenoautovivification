# NAME

Perl::Critic::Policy::Variables::RequireNoAutovivification - Say no autovivification where a read through an element would create it.

# VERSION

version 0.001

# Perl::Critic::Policy::Variables::RequireNoAutovivification

A read through an element creates that element when it is missing.  Perl calls
this autovivification.  `$h{a}{b}` looks up `b` in the hash that `$h{a}`
refers to.  When `$h{a}` is not there, perl makes it an empty hash first, and
leaves it there after the read.  `exists $h{a}{b}` and `delete $h{a}{b}` do
the same.  The next `exists $h{a}`, `keys %h` or serialization of `%h` then
sees a key that nobody stored.

The `autovivification` pragma from CPAN stops it.  Say
`no autovivification;` in the scope of the read:

```perl
my $port = $config{server}{port};                              # reported

no autovivification;
my $port = $config{server}{port};                              # OK!

if ( exists $seen{$user}{$host} ) { ... }                      # reported
my @names = keys %{ $self->{children} };                       # reported

no autovivification;
if ( exists $seen{$user}{$host} ) { ... }                      # OK!
my @names = keys %{ $self->{children} };                       # OK!
```

Only a read through an element is reported.  A read of one level,
`$h{$k}`, creates nothing in `%h`, so a file without a read through an
element needs no pragma.  A store through an element creates the levels on
purpose, and is not reported:

```perl
$count{$user}{$host}++;                                        # OK!
push @{ $children{$parent} }, $child;                          # OK!
```

## PROHIBITED

Each of these, without a `no autovivification` in scope that covers it:

```perl
$h{a}{b}                  # two levels
$h{a}->{b}                # the same, with the arrow
$r->{a}[0]                # a reference, then two levels
$_[0]{a}                  # an element of @_ is the caller's variable
$$r{a}{b}                 # $r->{a}{b}
${ $h{a} }{b}             # a block dereference of an element
@{ $h{a} }{qw(b c)}       # a slice of an element
$h{a}->@{qw(b c)}         # a postfix slice
$o->m->{a}{b}             # through what a method returned
@{ $h{a}{b} }             # a read through an element, inside a cast
keys %{ $h{a} }           # keys or values of an element
keys $h{a}->%*            # with a postfix cast
exists $h{a}{b}           # needs the 'exists' category
delete $h{a}{b}           # needs the 'delete' category
```

## ALLOWED

```perl
$h{a}                     # one level
$r->{a}                   # one level, through a reference
@{ $h{a} }                # strict refs dies here, it does not vivify
exists $h{a}              # one level
f()->{a}                  # one level of what a sub returned
( f() )[0]{k}             # a list slice is a copy
$h{a}{b} = 1              # an assignment
$h{a}{b}++                # and any other change
push @{ $h{a}{b} }, 1     # the first argument of push, pop and the like
\$h{a}{b}                 # a reference
local $h{a}{b}
f( $h{a}{b} )             # an argument of a sub or method, see CAVEATS
$h{a}{b}->m               # the invocant of a method
for ( @{ $h{a}{b} } ) {}  # a foreach list, and the list of map and grep
$h{a}{b} =~ s/x//         # a substitution or transliteration
chomp $h{a}{b}            # chomp, chop, undef, sprintf
```

## SCOPE AND CATEGORIES

`no autovivification` is lexical, like `no strict`.  It covers the rest of
the block that it is in, from the statement after it, and every block inside
that.  A `no autovivification` inside a sub covers only that sub.

Each read needs one category.  A read, and `keys` or `values`, needs
`fetch`.  `exists` needs `exists`, and `delete` needs `delete`.  Without
arguments, `no autovivification` turns on all three.  With arguments, it
turns on only the categories it names, so these do not cover a read:

```
no autovivification qw(store);     # store only
no autovivification 'warn';        # warn and strict alone turn on nothing
no autovivification ();            # empty parens do not call unimport
```

`warn` and `strict` with categories, such as
`no autovivification strict => 'fetch'`, turn those categories on.  Each
`no autovivification` adds to the categories in scope.  `use
autovivification` takes away the ones it names, or all of them without
arguments, until the end of its block.

The policy does not ask for `store`.  Under `store`, a store through a
missing level dies instead of creating it.  That changes what a correct
program does, where the other categories only stop a read from writing.

## CAVEATS

Perl passes the arguments of a sub or method, a `foreach` list, and the list
of `map` and `grep` as aliases, so it creates the levels above them as it
does for a store.  The default categories do not stop that, so the policy does
not report it.  Copy the value into a variable first:

```perl
my $port = $config{server}{port};
connect_to($port);
```

An argument that the policy cannot read, such as
`no autovivification @categories`, counts as no arguments.

The policy does not read inside strings.  `"$h{a}{b}"` creates `$h{a}` as
well.

A read of one level through a reference, `$r->{a}`, makes `$r` a
reference to an empty hash when `$r` is undefined.  The policy does not report
it: the variable that changes is the one that the code reads through, and the
code is about to find out what it holds.

# CONFIGURATION

`equivalent_modules` names modules that turn the default categories on in
the scope of the code that uses them, the way some bundles of pragmas do.  A
`use` of one of them counts as `no autovivification;`.

```
[Variables::RequireNoAutovivification]
equivalent_modules = My::Kit
```

## METHODS

### supported\_parameters

### default\_severity

### default\_themes

### applies\_to

### violates

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/teodesian/perl-critic-policy-requirenoautovivification/issues](https://github.com/teodesian/perl-critic-policy-requirenoautovivification/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# CONTRIBUTOR

troglodyne-bot <morpheus@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
