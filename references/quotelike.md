## Quote and Quote-like Operators

While we usually think of quotes as literal values, in Perl they
function as operators, providing various kinds of interpolating and
pattern matching capabilities.  Perl provides customary quote characters
for these behaviors, but also provides a way for you to choose your
quote character for any of them.  In the following table, a `{}` represents
any pair of delimiters you choose.

       Customary  Generic        Meaning        Interpolates
           ''       q{}          Literal             no
           ""      qq{}          Literal             yes
           ``      qx{}          Command             yes*
                   qw{}         Word list            no
           //       m{}       Pattern match          yes*
                   qr{}          Pattern             yes*
                    s{}{}      Substitution          yes*
                   tr{}{}    Transliteration         no (but see below)
                    y{}{}    Transliteration         no (but see below)
           <<EOF                 here-doc            yes*
    
           * unless the delimiter is ''.
    

Non-bracketing delimiters use the same character fore and aft, but the four
sorts of ASCII brackets (round, angle, square, curly) all nest, which means
that

       q{foo{bar}baz}
    

is the same as

       'foo{bar}baz'
    

Note, however, that this does not always work for quoting Perl code:

       $s = q{ if($x eq "}") ... }; # WRONG
    

is a syntax error.  The `[Text::Balanced](https://metacpan.org/pod/Text::Balanced)` module (standard as of v5.8,
and from CPAN before then) is able to do this properly.

There can (and in some cases, must) be whitespace between the operator
and the quoting
characters, except when `#` is being used as the quoting character.
`q#foo#` is parsed as the string `foo`, while `q #foo#` is the
operator `q` followed by a comment.  Its argument will be taken
from the next line.  This allows you to write:

       s {foo}  # Replace foo
         {bar}  # with bar.
    

The cases where whitespace must be used are when the quoting character
is a word character (meaning it matches `/\w/`):

       q XfooX # Works: means the string 'foo'
       qXfooX  # WRONG!
    

The following escape sequences are available in constructs that interpolate,
and in transliterations whose delimiters aren't single quotes (`"'"`).
           


       Sequence     Note  Description
       \t                  tab               (HT, TAB)
       \n                  newline           (NL)
       \r                  return            (CR)
       \f                  form feed         (FF)
       \b                  backspace         (BS)
       \a                  alarm (bell)      (BEL)
       \e                  escape            (ESC)
       \x{263A}     [1,8]  hex char          (example: SMILEY)
       \x1b         [2,8]  restricted range hex char (example: ESC)
       \N{name}     [3]    named Unicode character or character sequence
       \N{U+263D}   [4,8]  Unicode character (example: FIRST QUARTER MOON)
       \c[          [5]    control char      (example: chr(27))
       \o{23072}    [6,8]  octal char        (example: SMILEY)
       \033         [7,8]  restricted range octal char  (example: ESC)
    

- \[1\]

    The result is the character specified by the hexadecimal number between
    the braces.  See ["\[8\]"](#8) below for details on which character.

    Only hexadecimal digits are valid between the braces.  If an invalid
    character is encountered, a warning will be issued and the invalid
    character and all subsequent characters (valid or invalid) within the
    braces will be discarded.

    If there are no valid digits between the braces, the generated character is
    the NULL character (`\x{00}`).  However, an explicit empty brace (`\x{}`)
    will not cause a warning (currently).

- \[2\]

    The result is the character specified by the hexadecimal number in the range
    0x00 to 0xFF.  See ["\[8\]"](#8) below for details on which character.

    Only hexadecimal digits are valid following `\x`.  When `\x` is followed
    by fewer than two valid digits, any valid digits will be zero-padded.  This
    means that `\x7` will be interpreted as `\x07`, and a lone `"\x"` will be
    interpreted as `\x00`.  Except at the end of a string, having fewer than
    two valid digits will result in a warning.  Note that although the warning
    says the illegal character is ignored, it is only ignored as part of the
    escape and will still be used as the subsequent character in the string.
    For example:

         Original    Result    Warns?
         "\x7"       "\x07"    no
         "\x"        "\x00"    no
         "\x7q"      "\x07q"   yes
         "\xq"       "\x00q"   yes
        

- \[3\]

    The result is the Unicode character or character sequence given by _name_.
    See [charnames](https://metacpan.org/pod/charnames).

- \[4\]

    `\N{U+_hexadecimal number_}` means the Unicode character whose Unicode code
    point is _hexadecimal number_.

- \[5\]

    The character following `\c` is mapped to some other character as shown in the
    table:

        Sequence   Value
          \c@      chr(0)
          \cA      chr(1)
          \ca      chr(1)
          \cB      chr(2)
          \cb      chr(2)
          ...
          \cZ      chr(26)
          \cz      chr(26)
          \c[      chr(27)
                            # See below for chr(28)
          \c]      chr(29)
          \c^      chr(30)
          \c_      chr(31)
          \c?      chr(127) # (on ASCII platforms; see below for link to
                            #  EBCDIC discussion)
        

    In other words, it's the character whose code point has had 64 xor'd with
    its uppercase.  `\c?` is DELETE on ASCII platforms because
    `ord("?") ^ 64` is 127, and
    `\c@` is NULL because the ord of `"@"` is 64, so xor'ing 64 itself produces 0.

    Also, `\c\_X_` yields ` chr(28) . "_X_"` for any _X_, but cannot come at the
    end of a string, because the backslash would be parsed as escaping the end
    quote.

    On ASCII platforms, the resulting characters from the list above are the
    complete set of ASCII controls.  This isn't the case on EBCDIC platforms; see
    ["OPERATOR DIFFERENCES" in perlebcdic](https://metacpan.org/pod/perlebcdic#OPERATOR-DIFFERENCES) for a full discussion of the
    differences between these for ASCII versus EBCDIC platforms.

    Use of any other character following the `"c"` besides those listed above is
    discouraged, and as of Perl v5.20, the only characters actually allowed
    are the printable ASCII ones, minus the left brace `"{"`.  What happens
    for any of the allowed other characters is that the value is derived by
    xor'ing with the seventh bit, which is 64, and a warning raised if
    enabled.  Using the non-allowed characters generates a fatal error.

    To get platform independent controls, you can use `\N{...}`.

- \[6\]

    The result is the character specified by the octal number between the braces.
    See ["\[8\]"](#8) below for details on which character.

    If a character that isn't an octal digit is encountered, a warning is raised,
    and the value is based on the octal digits before it, discarding it and all
    following characters up to the closing brace.  It is a fatal error if there are
    no octal digits at all.

- \[7\]

    The result is the character specified by the three-digit octal number in the
    range 000 to 777 (but best to not use above 077, see next paragraph).  See
    ["\[8\]"](#8) below for details on which character.

    Some contexts allow 2 or even 1 digit, but any usage without exactly
    three digits, the first being a zero, may give unintended results.  (For
    example, in a regular expression it may be confused with a backreference;
    see ["Octal escapes" in perlrebackslash](https://metacpan.org/pod/perlrebackslash#Octal-escapes).)  Starting in Perl 5.14, you may
    use `\o{}` instead, which avoids all these problems.  Otherwise, it is best to
    use this construct only for ordinals `\077` and below, remembering to pad to
    the left with zeros to make three digits.  For larger ordinals, either use
    `\o{}`, or convert to something else, such as to hex and use `\N{U+}`
    (which is portable between platforms with different character sets) or
    `\x{}` instead.

- \[8\]

    Several constructs above specify a character by a number.  That number
    gives the character's position in the character set encoding (indexed from 0).
    This is called synonymously its ordinal, code position, or code point.  Perl
    works on platforms that have a native encoding currently of either ASCII/Latin1
    or EBCDIC, each of which allow specification of 256 characters.  In general, if
    the number is 255 (0xFF, 0377) or below, Perl interprets this in the platform's
    native encoding.  If the number is 256 (0x100, 0400) or above, Perl interprets
    it as a Unicode code point and the result is the corresponding Unicode
    character.  For example `\x{50}` and `\o{120}` both are the number 80 in
    decimal, which is less than 256, so the number is interpreted in the native
    character set encoding.  In ASCII the character in the 80th position (indexed
    from 0) is the letter `"P"`, and in EBCDIC it is the ampersand symbol `"&"`.
    `\x{100}` and `\o{400}` are both 256 in decimal, so the number is interpreted
    as a Unicode code point no matter what the native encoding is.  The name of the
    character in the 256th position (indexed by 0) in Unicode is
    `LATIN CAPITAL LETTER A WITH MACRON`.

    An exception to the above rule is that `\N{U+_hex number_}` is
    always interpreted as a Unicode code point, so that `\N{U+0050}` is `"P"` even
    on EBCDIC platforms.

**NOTE**: Unlike C and other languages, Perl has no `\v` escape sequence for
the vertical tab (VT, which is 11 in both ASCII and EBCDIC), but you may
use `\N{VT}`, `\ck`, `\N{U+0b}`, or `\x0b`.  (`\v`
does have meaning in regular expression patterns in Perl, see [perlre](https://metacpan.org/pod/perlre).)

The following escape sequences are available in constructs that interpolate,
but not in transliterations.
      

       \l          lowercase next character only
       \u          titlecase (not uppercase!) next character only
       \L          lowercase all characters till \E or end of string
       \U          uppercase all characters till \E or end of string
       \F          foldcase all characters till \E or end of string
       \Q          quote (disable) pattern metacharacters till \E or
                   end of string
       \E          end either case modification or quoted section
                   (whichever was last seen)
    

See ["quotemeta" in perlfunc](https://metacpan.org/pod/perlfunc#quotemeta) for the exact definition of characters that
are quoted by `\Q`.

`\L`, `\U`, `\F`, and `\Q` can stack, in which case you need one
`\E` for each.  For example:

    say"This \Qquoting \ubusiness \Uhere isn't quite\E done yet,\E is it?";
    This quoting\ Business\ HERE\ ISN\'T\ QUITE\ done\ yet\, is it?
    

If a `use locale` form that includes `LC_CTYPE` is in effect (see
[perllocale](https://metacpan.org/pod/perllocale)), the case map used by `\l`, `\L`, `\u`, and `\U` is
taken from the current locale.  If Unicode (for example, `\N{}` or code
points of 0x100 or beyond) is being used, the case map used by `\l`,
`\L`, `\u`, and `\U` is as defined by Unicode.  That means that
case-mapping a single character can sometimes produce a sequence of
several characters.
Under `use locale`, `\F` produces the same results as `\L`
for all locales but a UTF-8 one, where it instead uses the Unicode
definition.

All systems use the virtual `"\n"` to represent a line terminator,
called a "newline".  There is no such thing as an unvarying, physical
newline character.  It is only an illusion that the operating system,
device drivers, C libraries, and Perl all conspire to preserve.  Not all
systems read `"\r"` as ASCII CR and `"\n"` as ASCII LF.  For example,
on the ancient Macs (pre-MacOS X) of yesteryear, these used to be reversed,
and on systems without a line terminator,
printing `"\n"` might emit no actual data.  In general, use `"\n"` when
you mean a "newline" for your system, but use the literal ASCII when you
need an exact character.  For example, most networking protocols expect
and prefer a CR+LF (`"\015\012"` or `"\cM\cJ"`) for line terminators,
and although they often accept just `"\012"`, they seldom tolerate just
`"\015"`.  If you get in the habit of using `"\n"` for networking,
you may be burned some day.
   
  

For constructs that do interpolate, variables beginning with "`$`"
or "`@`" are interpolated.  Subscripted variables such as `$a[3]` or
`$href->{key}[0]` are also interpolated, as are array and hash slices.
But method calls such as `$obj->meth` are not.

Interpolating an array or slice interpolates the elements in order,
separated by the value of `$"`, so is equivalent to interpolating
`join $", @array`.  "Punctuation" arrays such as `@*` are usually
interpolated only if the name is enclosed in braces `@{*}`, but the
arrays `@_`, `@+`, and `@-` are interpolated even without braces.

For double-quoted strings, the quoting from `\Q` is applied after
interpolation and escapes are processed.

       "abc\Qfoo\tbar$s\Exyz"
    

is equivalent to

       "abc" . quotemeta("foo\tbar$s") . "xyz"
    

For the pattern of regex operators (`qr//`, `m//` and `s///`),
the quoting from `\Q` is applied after interpolation is processed,
but before escapes are processed.  This allows the pattern to match
literally (except for `$` and `@`).  For example, the following matches:

       '\s\t' =~ /\Q\s\t/
    

Because `$` or `@` trigger interpolation, you'll need to use something
like `/\Quser\E\@\Qhost/` to match them literally.

Patterns are subject to an additional level of interpretation as a
regular expression.  This is done as a second pass, after variables are
interpolated, so that regular expressions may be incorporated into the
pattern from the variables.  If this is not what you want, use `\Q` to
interpolate a variable literally.

Apart from the behavior described above, Perl does not expand
multiple levels of interpolation.  In particular, contrary to the
expectations of shell programmers, back-quotes do _NOT_ interpolate
within double quotes, nor do single quotes impede evaluation of
variables when used within double quotes.

## Regexp Quote-Like Operators


Here are the quote-like operators that apply to pattern
matching and related activities.

- `qr/_STRING_/msixpodualn`
      

    This operator quotes (and possibly compiles) its _STRING_ as a regular
    expression.  _STRING_ is interpolated the same way as _PATTERN_
    in `m/_PATTERN_/`.  If `"'"` is used as the delimiter, no variable
    interpolation is done.  Returns a Perl value which may be used instead of the
    corresponding `/_STRING_/msixpodualn` expression.  The returned value is a
    normalized version of the original pattern.  It magically differs from
    a string containing the same characters: `ref(qr/x/)` returns "Regexp";
    however, dereferencing it is not well defined (you currently get the
    normalized version of the original pattern, but this may change).

    For example,

           $rex = qr/my.STRING/is;
           print $rex;                 # prints (?si-xm:my.STRING)
           s/$rex/foo/;
        

    is equivalent to

           s/my.STRING/foo/is;
        

    The result may be used as a subpattern in a match:

           $re = qr/$pattern/;
           $string =~ /foo${re}bar/;   # can be interpolated in other
                                       # patterns
           $string =~ $re;             # or used standalone
           $string =~ /$re/;           # or this way
        

    Since Perl may compile the pattern at the moment of execution of the `qr()`
    operator, using `qr()` may have speed advantages in some situations,
    notably if the result of `qr()` is used standalone:

           sub match {
               my $patterns = shift;
               my @compiled = map qr/$_/i, @$patterns;
               grep {
                   my $success = 0;
                   foreach my $pat (@compiled) {
                       $success = 1, last if /$pat/;
                   }
                   $success;
               } @_;
           }
        

    Precompilation of the pattern into an internal representation at
    the moment of `qr()` avoids the need to recompile the pattern every
    time a match `/$pat/` is attempted.  (Perl has many other internal
    optimizations, but none would be triggered in the above example if
    we did not use `qr()` operator.)

    Options (specified by the following modifiers) are:

           m   Treat string as multiple lines.
           s   Treat string as single line. (Make . match a newline)
           i   Do case-insensitive pattern matching.
           x   Use extended regular expressions; specifying two
               x's means \t and the SPACE character are ignored within
               square-bracketed character classes
           p   When matching preserve a copy of the matched string so
               that ${^PREMATCH}, ${^MATCH}, ${^POSTMATCH} will be
               defined (ignored starting in v5.20) as these are always
               defined starting in that release
           o   Compile pattern only once.
           a   ASCII-restrict: Use ASCII for \d, \s, \w and [[:posix:]]
               character classes; specifying two a's adds the further
               restriction that no ASCII character will match a
               non-ASCII one under /i.
           l   Use the current run-time locale's rules.
           u   Use Unicode rules.
           d   Use Unicode or native charset, as in 5.12 and earlier.
           n   Non-capture mode. Don't let () fill in $1, $2, etc...
        

    If a precompiled pattern is embedded in a larger pattern then the effect
    of `"msixpluadn"` will be propagated appropriately.  The effect that the
    `/o` modifier has is not propagated, being restricted to those patterns
    explicitly using it.

    The `/a`, `/d`, `/l`, and `/u` modifiers (added in Perl 5.14)
    control the character set rules, but `/a` is the only one you are likely
    to want to specify explicitly; the other three are selected
    automatically by various pragmas.

    See [perlre](https://metacpan.org/pod/perlre) for additional information on valid syntax for _STRING_, and
    for a detailed look at the semantics of regular expressions.  In
    particular, all modifiers except the largely obsolete `/o` are further
    explained in ["Modifiers" in perlre](https://metacpan.org/pod/perlre#Modifiers).  `/o` is described in the next section.

- `m/_PATTERN_/msixpodualngc`
 
   
       
- `/_PATTERN_/msixpodualngc`

    Searches a string for a pattern match, and in scalar context returns
    true if it succeeds, false if it fails.  If no string is specified
    via the `=~` or `!~` operator, the `$_` string is searched.  (The
    string specified with `=~` need not be an lvalue--it may be the
    result of an expression evaluation, but remember the `=~` binds
    rather tightly.)  See also [perlre](https://metacpan.org/pod/perlre).

    Options are as described in `qr//` above; in addition, the following match
    process modifiers are available:

        g  Match globally, i.e., find all occurrences.
        c  Do not reset search position on a failed match when /g is
           in effect.
        

    If `"/"` is the delimiter then the initial `m` is optional.  With the `m`
    you can use any pair of non-whitespace (ASCII) characters
    as delimiters.  This is particularly useful for matching path names
    that contain `"/"`, to avoid LTS (leaning toothpick syndrome).  If `"?"` is
    the delimiter, then a match-only-once rule applies,
    described in `m?_PATTERN_?` below.  If `"'"` (single quote) is the delimiter,
    no variable interpolation is performed on the _PATTERN_.
    When using a delimiter character valid in an identifier, whitespace is required
    after the `m`.

    _PATTERN_ may contain variables, which will be interpolated
    every time the pattern search is evaluated, except
    for when the delimiter is a single quote.  (Note that `$(`, `$)`, and
    `$|` are not interpolated because they look like end-of-string tests.)
    Perl will not recompile the pattern unless an interpolated
    variable that it contains changes.  You can force Perl to skip the
    test and never recompile by adding a `/o` (which stands for "once")
    after the trailing delimiter.
    Once upon a time, Perl would recompile regular expressions
    unnecessarily, and this modifier was useful to tell it not to do so, in the
    interests of speed.  But now, the only reasons to use `/o` are one of:

    1. The variables are thousands of characters long and you know that they
    don't change, and you need to wring out the last little bit of speed by
    having Perl skip testing for that.  (There is a maintenance penalty for
    doing this, as mentioning `/o` constitutes a promise that you won't
    change the variables in the pattern.  If you do change them, Perl won't
    even notice.)
    2. you want the pattern to use the initial values of the variables
    regardless of whether they change or not.  (But there are saner ways
    of accomplishing this than using `/o`.)
    3. If the pattern contains embedded code, such as

               use re 'eval';
               $code = 'foo(?{ $x })';
               /$code/
            

        then perl will recompile each time, even though the pattern string hasn't
        changed, to ensure that the current value of `$x` is seen each time.
        Use `/o` if you want to avoid this.

    The bottom line is that using `/o` is almost never a good idea.

- The empty pattern `//`

    If the _PATTERN_ evaluates to the empty string, the last
    _successfully_ matched regular expression is used instead.  In this
    case, only the `g` and `c` flags on the empty pattern are honored;
    the other flags are taken from the original pattern.  If no match has
    previously succeeded, this will (silently) act instead as a genuine
    empty pattern (which will always match).

    Note that it's possible to confuse Perl into thinking `//` (the empty
    regex) is really `//` (the defined-or operator).  Perl is usually pretty
    good about this, but some pathological cases might trigger this, such as
    `$x///` (is that `($x) / (//)` or `$x // /`?) and `print $fh //`
    (`print $fh(//` or `print($fh //`?).  In all of these examples, Perl
    will assume you meant defined-or.  If you meant the empty regex, just
    use parentheses or spaces to disambiguate, or even prefix the empty
    regex with an `m` (so `//` becomes `m//`).

- Matching in list context

    If the `/g` option is not used, `m//` in list context returns a
    list consisting of the subexpressions matched by the parentheses in the
    pattern, that is, (`$1`, `$2`, `$3`...)  (Note that here `$1` etc. are
    also set).  When there are no parentheses in the pattern, the return
    value is the list `(1)` for success.
    With or without parentheses, an empty list is returned upon failure.

    Examples:

        open(TTY, "+</dev/tty")
           || die "can't access /dev/tty: $!";
        
        <TTY> =~ /^y/i && foo(); # do foo if desired
        
        if (/Version: *([0-9.]*)/) { $version = $1; }
        
        next if m#^/usr/spool/uucp#;
        
        # poor man's grep
        $arg = shift;
        while (<>) {
           print if /$arg/o; # compile only once (no longer needed!)
        }
        
        if (($F1, $F2, $Etc) = ($foo =~ /^(\S+)\s+(\S+)\s*(.*)/))
        

    This last example splits `$foo` into the first two words and the
    remainder of the line, and assigns those three fields to `$F1`, `$F2`, and
    `$Etc`.  The conditional is true if any variables were assigned; that is,
    if the pattern matched.

    The `/g` modifier specifies global pattern matching--that is,
    matching as many times as possible within the string.  How it behaves
    depends on the context.  In list context, it returns a list of the
    substrings matched by any capturing parentheses in the regular
    expression.  If there are no parentheses, it returns a list of all
    the matched strings, as if there were parentheses around the whole
    pattern.

    In scalar context, each execution of `m//g` finds the next match,
    returning true if it matches, and false if there is no further match.
    The position after the last match can be read or set using the `pos()`
    function; see ["pos" in perlfunc](https://metacpan.org/pod/perlfunc#pos).  A failed match normally resets the
    search position to the beginning of the string, but you can avoid that
    by adding the `/c` modifier (for example, `m//gc`).  Modifying the target
    string also resets the search position.

- `\G _assertion_`

    You can intermix `m//g` matches with `m/\G.../g`, where `\G` is a
    zero-width assertion that matches the exact position where the
    previous `m//g`, if any, left off.  Without the `/g` modifier, the
    `\G` assertion still anchors at `pos()` as it was at the start of
    the operation (see ["pos" in perlfunc](https://metacpan.org/pod/perlfunc#pos)), but the match is of course only
    attempted once.  Using `\G` without `/g` on a target string that has
    not previously had a `/g` match applied to it is the same as using
    the `\A` assertion to match the beginning of the string.  Note also
    that, currently, `\G` is only properly supported when anchored at the
    very beginning of the pattern.

    Examples:

           # list context
           ($one,$five,$fifteen) = (`uptime` =~ /(\d+\.\d+)/g);
        
           # scalar context
           local $/ = "";
           while ($paragraph = <>) {
               while ($paragraph =~ /\p{Ll}['")]*[.!?]+['")]*\s/g) {
                   $sentences++;
               }
           }
           say $sentences;
        

    Here's another way to check for sentences in a paragraph:

        my $sentence_rx = qr{
           (?: (?<= ^ ) | (?<= \s ) )  # after start-of-string or
                                       # whitespace
           \p{Lu}                      # capital letter
           .*?                         # a bunch of anything
           (?<= \S )                   # that ends in non-
                                       # whitespace
           (?<! \b [DMS]r  )           # but isn't a common abbr.
           (?<! \b Mrs )
           (?<! \b Sra )
           (?<! \b St  )
           [.?!]                       # followed by a sentence
                                       # ender
           (?= $ | \s )                # in front of end-of-string
                                       # or whitespace
        }sx;
        local $/ = "";
        while (my $paragraph = <>) {
           say "NEW PARAGRAPH";
           my $count = 0;
           while ($paragraph =~ /($sentence_rx)/g) {
               printf "\tgot sentence %d: <%s>\n", ++$count, $1;
           }
        }
        

    Here's how to use `m//gc` with `\G`:

           $_ = "ppooqppqq";
           while ($i++ < 2) {
               print "1: '";
               print $1 while /(o)/gc; print "', pos=", pos, "\n";
               print "2: '";
               print $1 if /\G(q)/gc;  print "', pos=", pos, "\n";
               print "3: '";
               print $1 while /(p)/gc; print "', pos=", pos, "\n";
           }
           print "Final: '$1', pos=",pos,"\n" if /\G(.)/;
        

    The last example should print:

           1: 'oo', pos=4
           2: 'q', pos=5
           3: 'pp', pos=7
           1: '', pos=7
           2: 'q', pos=8
           3: '', pos=8
           Final: 'q', pos=8
        

    Notice that the final match matched `q` instead of `p`, which a match
    without the `\G` anchor would have done.  Also note that the final match
    did not update `pos`.  `pos` is only updated on a `/g` match.  If the
    final match did indeed match `p`, it's a good bet that you're running an
    ancient (pre-5.6.0) version of Perl.

    A useful idiom for `lex`-like scanners is `/\G.../gc`.  You can
    combine several regexps like this to process a string part-by-part,
    doing different actions depending on which regexp matched.  Each
    regexp tries to match where the previous one leaves off.

        $_ = <<'EOL';
           $url = URI::URL->new( "http://example.com/" );
           die if $url eq "xXx";
        EOL
        
        LOOP: {
            print(" digits"),       redo LOOP if /\G\d+\b[,.;]?\s*/gc;
            print(" lowercase"),    redo LOOP
                                           if /\G\p{Ll}+\b[,.;]?\s*/gc;
            print(" UPPERCASE"),    redo LOOP
                                           if /\G\p{Lu}+\b[,.;]?\s*/gc;
            print(" Capitalized"),  redo LOOP
                                     if /\G\p{Lu}\p{Ll}+\b[,.;]?\s*/gc;
            print(" MiXeD"),        redo LOOP if /\G\pL+\b[,.;]?\s*/gc;
            print(" alphanumeric"), redo LOOP
                                   if /\G[\p{Alpha}\pN]+\b[,.;]?\s*/gc;
            print(" line-noise"),   redo LOOP if /\G\W+/gc;
            print ". That's all!\n";
        }
        

    Here is the output (split into several lines):

        line-noise lowercase line-noise UPPERCASE line-noise UPPERCASE
        line-noise lowercase line-noise lowercase line-noise lowercase
        lowercase line-noise lowercase lowercase line-noise lowercase
        lowercase line-noise MiXeD line-noise. That's all!
        

- `m?_PATTERN_?msixpodualngc`
 

    This is just like the `m/_PATTERN_/` search, except that it matches
    only once between calls to the `reset()` operator.  This is a useful
    optimization when you want to see only the first occurrence of
    something in each file of a set of files, for instance.  Only `m??`
    patterns local to the current package are reset.

           while (<>) {
               if (m?^$?) {
                                   # blank line between header and body
               }
           } continue {
               reset if eof;       # clear m?? status for next file
           }
        

    Another example switched the first "latin1" encoding it finds
    to "utf8" in a pod file:

           s//utf8/ if m? ^ =encoding \h+ \K latin1 ?x;
        

    The match-once behavior is controlled by the match delimiter being
    `?`; with any other delimiter this is the normal `m//` operator.

    In the past, the leading `m` in `m?_PATTERN_?` was optional, but omitting it
    would produce a deprecation warning.  As of v5.22.0, omitting it produces a
    syntax error.  If you encounter this construct in older code, you can just add
    `m`.

- `s/_PATTERN_/_REPLACEMENT_/msixpodualngcer`
    
          

    Searches a string for a pattern, and if found, replaces that pattern
    with the replacement text and returns the number of substitutions
    made.  Otherwise it returns false (a value that is both an empty string (`""`)
    and numeric zero (`0`) as described in ["Relational Operators"](#relational-operators)).

    If the `/r` (non-destructive) option is used then it runs the
    substitution on a copy of the string and instead of returning the
    number of substitutions, it returns the copy whether or not a
    substitution occurred.  The original string is never changed when
    `/r` is used.  The copy will always be a plain string, even if the
    input is an object or a tied variable.

    If no string is specified via the `=~` or `!~` operator, the `$_`
    variable is searched and modified.  Unless the `/r` option is used,
    the string specified must be a scalar variable, an array element, a
    hash element, or an assignment to one of those; that is, some sort of
    scalar lvalue.

    If the delimiter chosen is a single quote, no variable interpolation is
    done on either the _PATTERN_ or the _REPLACEMENT_.  Otherwise, if the
    _PATTERN_ contains a `$` that looks like a variable rather than an
    end-of-string test, the variable will be interpolated into the pattern
    at run-time.  If you want the pattern compiled only once the first time
    the variable is interpolated, use the `/o` option.  If the pattern
    evaluates to the empty string, the last successfully executed regular
    expression is used instead.  See [perlre](https://metacpan.org/pod/perlre) for further explanation on these.

    Options are as with `m//` with the addition of the following replacement
    specific options:

           e   Evaluate the right side as an expression.
           ee  Evaluate the right side as a string then eval the
               result.
           r   Return substitution and leave the original string
               untouched.
        

    Any non-whitespace delimiter may replace the slashes.  Add space after
    the `s` when using a character allowed in identifiers.  If single quotes
    are used, no interpretation is done on the replacement string (the `/e`
    modifier overrides this, however).  Note that Perl treats backticks
    as normal delimiters; the replacement text is not evaluated as a command.
    If the _PATTERN_ is delimited by bracketing quotes, the _REPLACEMENT_ has
    its own pair of quotes, which may or may not be bracketing quotes, for example,
    `s(foo)(bar)` or `s<foo>/bar/`.  A `/e` will cause the
    replacement portion to be treated as a full-fledged Perl expression
    and evaluated right then and there.  It is, however, syntax checked at
    compile-time.  A second `e` modifier will cause the replacement portion
    to be `eval`ed before being run as a Perl expression.

    Examples:

           s/\bgreen\b/mauve/g;              # don't change wintergreen
        
           $path =~ s|/usr/bin|/usr/local/bin|;
        
           s/Login: $foo/Login: $bar/; # run-time pattern
        
           ($foo = $bar) =~ s/this/that/;      # copy first, then
                                               # change
           ($foo = "$bar") =~ s/this/that/;    # convert to string,
                                               # copy, then change
           $foo = $bar =~ s/this/that/r;       # Same as above using /r
           $foo = $bar =~ s/this/that/r
                       =~ s/that/the other/r;  # Chained substitutes
                                               # using /r
           @foo = map { s/this/that/r } @bar   # /r is very useful in
                                               # maps
        
           $count = ($paragraph =~ s/Mister\b/Mr./g);  # get change-cnt
        
           $_ = 'abc123xyz';
           s/\d+/$&*2/e;           # yields 'abc246xyz'
           s/\d+/sprintf("%5d",$&)/e;      # yields 'abc  246xyz'
           s/\w/$& x 2/eg;         # yields 'aabbcc  224466xxyyzz'
        
           s/%(.)/$percent{$1}/g;      # change percent escapes; no /e
           s/%(.)/$percent{$1} || $&/ge;   # expr now, so /e
           s/^=(\w+)/pod($1)/ge;       # use function call
        
           $_ = 'abc123xyz';
           $x = s/abc/def/r;           # $x is 'def123xyz' and
                                       # $_ remains 'abc123xyz'.
        
           # expand variables in $_, but dynamics only, using
           # symbolic dereferencing
           s/\$(\w+)/${$1}/g;
        
           # Add one to the value of any numbers in the string
           s/(\d+)/1 + $1/eg;
        
           # Titlecase words in the last 30 characters only
           substr($str, -30) =~ s/\b(\p{Alpha}+)\b/\u\L$1/g;
        
           # This will expand any embedded scalar variable
           # (including lexicals) in $_ : First $1 is interpolated
           # to the variable name, and then evaluated
           s/(\$\w+)/$1/eeg;
        
           # Delete (most) C comments.
           $program =~ s {
               /\*     # Match the opening delimiter.
               .*?     # Match a minimal number of characters.
               \*/     # Match the closing delimiter.
           } []gsx;
        
           s/^\s*(.*?)\s*$/$1/;        # trim whitespace in $_,
                                       # expensively
        
           for ($variable) {           # trim whitespace in $variable,
                                       # cheap
               s/^\s+//;
               s/\s+$//;
           }
        
           s/([^ ]*) *([^ ]*)/$2 $1/;  # reverse 1st two fields
        
           $foo !~ s/A/a/g;    # Lowercase all A's in $foo; return
                               # 0 if any were found and changed;
                               # otherwise return 1
        

    Note the use of `$` instead of `\` in the last example.  Unlike
    **sed**, we use the \\<_digit_> form only in the left hand side.
    Anywhere else it's $<_digit_>.

    Occasionally, you can't use just a `/g` to get all the changes
    to occur that you might want.  Here are two common cases:

           # put commas in the right places in an integer
           1 while s/(\d)(\d\d\d)(?!\d)/$1,$2/g;
        
           # expand tabs to 8-column spacing
           1 while s/\t+/' ' x (length($&)*8 - length($`)%8)/e;
        

## Quote-Like Operators


- `q/_STRING_/`
   
- `'_STRING_'`

    A single-quoted, literal string.  A backslash represents a backslash
    unless followed by the delimiter or another backslash, in which case
    the delimiter or backslash is interpolated.

           $foo = q!I said, "You said, 'She said it.'"!;
           $bar = q('This is it.');
           $baz = '\n';                # a two-character string
        

- `qq/_STRING_/`
   
- "_STRING_"

    A double-quoted, interpolated string.

           $_ .= qq
            (*** The previous line contains the naughty word "$1".\n)
                       if /\b(tcl|java|python)\b/i;      # :-)
           $baz = "\n";                # a one-character string
        

- `qx/_STRING_/`
   
- `` `_STRING_` ``

    A string which is (possibly) interpolated and then executed as a
    system command with `/bin/sh` or its equivalent.  Shell wildcards,
    pipes, and redirections will be honored.  The collected standard
    output of the command is returned; standard error is unaffected.  In
    scalar context, it comes back as a single (potentially multi-line)
    string, or `undef` if the command failed.  In list context, returns a
    list of lines (however you've defined lines with `$/` or
    `$INPUT_RECORD_SEPARATOR`), or an empty list if the command failed.

    Because backticks do not affect standard error, use shell file descriptor
    syntax (assuming the shell supports this) if you care to address this.
    To capture a command's STDERR and STDOUT together:

           $output = `cmd 2>&1`;
        

    To capture a command's STDOUT but discard its STDERR:

           $output = `cmd 2>/dev/null`;
        

    To capture a command's STDERR but discard its STDOUT (ordering is
    important here):

           $output = `cmd 2>&1 1>/dev/null`;
        

    To exchange a command's STDOUT and STDERR in order to capture the STDERR
    but leave its STDOUT to come out the old STDERR:

           $output = `cmd 3>&1 1>&2 2>&3 3>&-`;
        

    To read both a command's STDOUT and its STDERR separately, it's easiest
    to redirect them separately to files, and then read from those files
    when the program is done:

           system("program args 1>program.stdout 2>program.stderr");
        

    The STDIN filehandle used by the command is inherited from Perl's STDIN.
    For example:

           open(SPLAT, "stuff")   || die "can't open stuff: $!";
           open(STDIN, "<&SPLAT") || die "can't dupe SPLAT: $!";
           print STDOUT `sort`;
        

    will print the sorted contents of the file named `"stuff"`.

    Using single-quote as a delimiter protects the command from Perl's
    double-quote interpolation, passing it on to the shell instead:

           $perl_info  = qx(ps $$);            # that's Perl's $$
           $shell_info = qx'ps $$';            # that's the new shell's $$
        

    How that string gets evaluated is entirely subject to the command
    interpreter on your system.  On most platforms, you will have to protect
    shell metacharacters if you want them treated literally.  This is in
    practice difficult to do, as it's unclear how to escape which characters.
    See [perlsec](https://metacpan.org/pod/perlsec) for a clean and safe example of a manual `fork()` and `exec()`
    to emulate backticks safely.

    On some platforms (notably DOS-like ones), the shell may not be
    capable of dealing with multiline commands, so putting newlines in
    the string may not get you what you want.  You may be able to evaluate
    multiple commands in a single line by separating them with the command
    separator character, if your shell supports that (for example, `;` on
    many Unix shells and `&` on the Windows NT `cmd` shell).

    Perl will attempt to flush all files opened for
    output before starting the child process, but this may not be supported
    on some platforms (see [perlport](https://metacpan.org/pod/perlport)).  To be safe, you may need to set
    `$|` (`$AUTOFLUSH` in `[English](https://metacpan.org/pod/English)`) or call the `autoflush()` method of
    `[IO::Handle](https://metacpan.org/pod/IO::Handle)` on any open handles.

    Beware that some command shells may place restrictions on the length
    of the command line.  You must ensure your strings don't exceed this
    limit after any necessary interpolations.  See the platform-specific
    release notes for more details about your particular environment.

    Using this operator can lead to programs that are difficult to port,
    because the shell commands called vary between systems, and may in
    fact not be present at all.  As one example, the `type` command under
    the POSIX shell is very different from the `type` command under DOS.
    That doesn't mean you should go out of your way to avoid backticks
    when they're the right way to get something done.  Perl was made to be
    a glue language, and one of the things it glues together is commands.
    Just understand what you're getting yourself into.

    Like `system`, backticks put the child process exit code in `$?`.
    If you'd like to manually inspect failure, you can check all possible
    failure modes by inspecting `$?` like this:

           if ($? == -1) {
               print "failed to execute: $!\n";
           }
           elsif ($? & 127) {
               printf "child died with signal %d, %s coredump\n",
                   ($? & 127),  ($? & 128) ? 'with' : 'without';
           }
           else {
               printf "child exited with value %d\n", $? >> 8;
           }
        

    Use the [open](https://metacpan.org/pod/open) pragma to control the I/O layers used when reading the
    output of the command, for example:

         use open IN => ":encoding(UTF-8)";
         my $x = `cmd-producing-utf-8`;
        

    `qx//` can also be called like a function with ["readpipe" in perlfunc](https://metacpan.org/pod/perlfunc#readpipe).

    See ["I/O Operators"](#i-o-operators) for more discussion.

- `qw/_STRING_/`
  

    Evaluates to a list of the words extracted out of _STRING_, using embedded
    whitespace as the word delimiters.  It can be understood as being roughly
    equivalent to:

           split(" ", q/STRING/);
        

    the differences being that it only splits on ASCII whitespace,
    generates a real list at compile time, and
    in scalar context it returns the last element in the list.  So
    this expression:

           qw(foo bar baz)
        

    is semantically equivalent to the list:

           "foo", "bar", "baz"
        

    Some frequently seen examples:

           use POSIX qw( setlocale localeconv )
           @EXPORT = qw( foo bar baz );
        

    A common mistake is to try to separate the words with commas or to
    put comments into a multi-line `qw`-string.  For this reason, the
    `use warnings` pragma and the **-w** switch (that is, the `$^W` variable)
    produces warnings if the _STRING_ contains the `","` or the `"#"` character.

- `tr/_SEARCHLIST_/_REPLACEMENTLIST_/cdsr`
     
- `y/_SEARCHLIST_/_REPLACEMENTLIST_/cdsr`

    Transliterates all occurrences of the characters found (or not found
    if the `/c` modifier is specified) in the search list with the
    positionally corresponding character in the replacement list, possibly
    deleting some, depending on the modifiers specified.  It returns the
    number of characters replaced or deleted.  If no string is specified via
    the `=~` or `!~` operator, the `$_` string is transliterated.

    For **sed** devotees, `y` is provided as a synonym for `tr`.

    If the `/r` (non-destructive) option is present, a new copy of the string
    is made and its characters transliterated, and this copy is returned no
    matter whether it was modified or not: the original string is always
    left unchanged.  The new copy is always a plain string, even if the input
    string is an object or a tied variable.

    Unless the `/r` option is used, the string specified with `=~` must be a
    scalar variable, an array element, a hash element, or an assignment to one
    of those; in other words, an lvalue.

    If the characters delimiting _SEARCHLIST_ and _REPLACEMENTLIST_
    are single quotes (`tr'_SEARCHLIST_'_REPLACEMENTLIST_'`), the only
    interpolation is removal of `\` from pairs of `\\`.

    Otherwise, a character range may be specified with a hyphen, so
    `tr/A-J/0-9/` does the same replacement as
    `tr/ACEGIBDFHJ/0246813579/`.

    If the _SEARCHLIST_ is delimited by bracketing quotes, the
    _REPLACEMENTLIST_ must have its own pair of quotes, which may or may
    not be bracketing quotes; for example, `tr[aeiouy][yuoiea]` or
    `tr(+\-*/)/ABCD/`.

    Characters may be literals, or (if the delimiters aren't single quotes)
    any of the escape sequences accepted in double-quoted strings.  But
    there is never any variable interpolation, so `"$"` and `"@"` are
    always treated as literals.  A hyphen at the beginning or end, or
    preceded by a backslash is also always considered a literal.  Escape
    sequence details are in [the table near the beginning of this
    section](#quote-and-quote-like-operators).

    Note that `tr` does **not** do regular expression character classes such as
    `\d` or `\pL`.  The `tr` operator is not equivalent to the `[tr(1)](http://man.he.net/man1/tr)`
    utility.  `tr[a-z][A-Z]` will uppercase the 26 letters "a" through "z",
    but for case changing not confined to ASCII, use
    [`lc`](https://metacpan.org/pod/perlfunc#lc), [`uc`](https://metacpan.org/pod/perlfunc#uc),
    [`lcfirst`](https://metacpan.org/pod/perlfunc#lcfirst), [`ucfirst`](https://metacpan.org/pod/perlfunc#ucfirst)
    (all documented in [perlfunc](https://metacpan.org/pod/perlfunc)), or the
    [substitution operator `s/_PATTERN_/_REPLACEMENT_/`](#s-pattern-replacement-msixpodualngcer)
    (with `\U`, `\u`, `\L`, and `\l` string-interpolation escapes in the
    _REPLACEMENT_ portion).

    Most ranges are unportable between character sets, but certain ones
    signal Perl to do special handling to make them portable.  There are two
    classes of portable ranges.  The first are any subsets of the ranges
    `A-Z`, `a-z`, and `0-9`, when expressed as literal characters.

         tr/h-k/H-K/
        

    capitalizes the letters `"h"`, `"i"`, `"j"`, and `"k"` and nothing
    else, no matter what the platform's character set is.  In contrast, all
    of

         tr/\x68-\x6B/\x48-\x4B/
         tr/h-\x6B/H-\x4B/
         tr/\x68-k/\x48-K/
        

    do the same capitalizations as the previous example when run on ASCII
    platforms, but something completely different on EBCDIC ones.

    The second class of portable ranges is invoked when one or both of the
    range's end points are expressed as `\N{...}`

        $string =~ tr/\N{U+20}-\N{U+7E}//d;
        

    removes from `$string` all the platform's characters which are
    equivalent to any of Unicode U+0020, U+0021, ... U+007D, U+007E.  This
    is a portable range, and has the same effect on every platform it is
    run on.  In this example, these are the ASCII
    printable characters.  So after this is run, `$string` has only
    controls and characters which have no ASCII equivalents.

    But, even for portable ranges, it is not generally obvious what is
    included without having to look things up in the manual.  A sound
    principle is to use only ranges that both begin from, and end at, either
    ASCII alphabetics of equal case (`b-e`, `B-E`), or digits (`1-4`).
    Anything else is unclear (and unportable unless `\N{...}` is used).  If
    in doubt, spell out the character sets in full.

    Options:

           c   Complement the SEARCHLIST.
           d   Delete found but unreplaced characters.
           r   Return the modified string and leave the original string
               untouched.
           s   Squash duplicate replaced characters.
        

    If the `/d` modifier is specified, any characters specified by
    _SEARCHLIST_  not found in _REPLACEMENTLIST_ are deleted.  (Note that
    this is slightly more flexible than the behavior of some **tr** programs,
    which delete anything they find in the _SEARCHLIST_, period.)

    If the `/s` modifier is specified, sequences of characters, all in a
    row, that were transliterated to the same character are squashed down to
    a single instance of that character.

        my $a = "aaaba"
        $a =~ tr/a/a/s     # $a now is "aba"
        

    If the `/d` modifier is used, the _REPLACEMENTLIST_ is always interpreted
    exactly as specified.  Otherwise, if the _REPLACEMENTLIST_ is shorter
    than the _SEARCHLIST_, the final character, if any, is replicated until
    it is long enough.  There won't be a final character if and only if the
    _REPLACEMENTLIST_ is empty, in which case _REPLACEMENTLIST_ is
    copied from _SEARCHLIST_.    An empty _REPLACEMENTLIST_ is useful
    for counting characters in a class, or for squashing character sequences
    in a class.

           tr/abcd//            tr/abcd/abcd/
           tr/abcd/AB/          tr/abcd/ABBB/
           tr/abcd//d           s/[abcd]//g
           tr/abcd/AB/d         (tr/ab/AB/ + s/[cd]//g)  - but run together
        

    If the `/c` modifier is specified, the characters to be transliterated
    are the ones NOT in _SEARCHLIST_, that is, it is complemented.  If
    `/d` and/or `/s` are also specified, they apply to the complemented
    _SEARCHLIST_.  Recall, that if _REPLACEMENTLIST_ is empty (except
    under `/d`) a copy of _SEARCHLIST_ is used instead.  That copy is made
    after complementing under `/c`.  _SEARCHLIST_ is sorted by code point
    order after complementing, and any _REPLACEMENTLIST_  is applied to
    that sorted result.  This means that under `/c`, the order of the
    characters specified in _SEARCHLIST_ is irrelevant.  This can
    lead to different results on EBCDIC systems if _REPLACEMENTLIST_
    contains more than one character, hence it is generally non-portable to
    use `/c` with such a _REPLACEMENTLIST_.

    Another way of describing the operation is this:
    If `/c` is specified, the _SEARCHLIST_ is sorted by code point order,
    then complemented.  If _REPLACEMENTLIST_ is empty and `/d` is not
    specified, _REPLACEMENTLIST_ is replaced by a copy of _SEARCHLIST_ (as
    modified under `/c`), and these potentially modified lists are used as
    the basis for what follows.  Any character in the target string that
    isn't in _SEARCHLIST_ is passed through unchanged.  Every other
    character in the target string is replaced by the character in
    _REPLACEMENTLIST_ that positionally corresponds to its mate in
    _SEARCHLIST_, except that under `/s`, the 2nd and following characters
    are squeezed out in a sequence of characters in a row that all translate
    to the same character.  If _SEARCHLIST_ is longer than
    _REPLACEMENTLIST_, characters in the target string that match a
    character in _SEARCHLIST_ that doesn't have a correspondence in
    _REPLACEMENTLIST_ are either deleted from the target string if `/d` is
    specified; or replaced by the final character in _REPLACEMENTLIST_ if
    `/d` isn't specified.

    Some examples:

        $ARGV[1] =~ tr/A-Z/a-z/;   # canonicalize to lower case ASCII
        
        $cnt = tr/*/*/;            # count the stars in $_
        $cnt = tr/*//;             # same thing
        
        $cnt = $sky =~ tr/*/*/;    # count the stars in $sky
        $cnt = $sky =~ tr/*//;     # same thing
        
        $cnt = $sky =~ tr/*//c;    # count all the non-stars in $sky
        $cnt = $sky =~ tr/*/*/c;   # same, but transliterate each non-star
                                   # into a star, leaving the already-stars
                                   # alone.  Afterwards, everything in $sky
                                   # is a star.
        
        $cnt = tr/0-9//;           # count the ASCII digits in $_
        
        tr/a-zA-Z//s;              # bookkeeper -> bokeper
        tr/o/o/s;                  # bookkeeper -> bokkeeper
        tr/oe/oe/s;                # bookkeeper -> bokkeper
        tr/oe//s;                  # bookkeeper -> bokkeper
        tr/oe/o/s;                 # bookkeeper -> bokkopor
        
        ($HOST = $host) =~ tr/a-z/A-Z/;
         $HOST = $host  =~ tr/a-z/A-Z/r; # same thing
        
        $HOST = $host =~ tr/a-z/A-Z/r   # chained with s///r
                      =~ s/:/ -p/r;
        
        tr/a-zA-Z/ /cs;                 # change non-alphas to single space
        
        @stripped = map tr/a-zA-Z/ /csr, @original;
                                        # /r with map
        
        tr [\200-\377]
           [\000-\177];                 # wickedly delete 8th bit
        
        $foo !~ tr/A/a/    # transliterate all the A's in $foo to 'a',
                           # return 0 if any were found and changed.
                           # Otherwise return 1
        

    If multiple transliterations are given for a character, only the
    first one is used:

        tr/AAA/XYZ/
        

    will transliterate any A to X.

    Because the transliteration table is built at compile time, neither
    the _SEARCHLIST_ nor the _REPLACEMENTLIST_ are subjected to double quote
    interpolation.  That means that if you want to use variables, you
    must use an `eval()`:

        eval "tr/$oldlist/$newlist/";
        die $@ if $@;
        
        eval "tr/$oldlist/$newlist/, 1" or die $@;
        

- `<<_EOF_`
   

    A line-oriented form of quoting is based on the shell "here-document"
    syntax.  Following a `<<` you specify a string to terminate
    the quoted material, and all lines following the current line down to
    the terminating string are the value of the item.

    Prefixing the terminating string with a `~` specifies that you
    want to use ["Indented Here-docs"](#indented-here-docs) (see below).

    The terminating string may be either an identifier (a word), or some
    quoted text.  An unquoted identifier works like double quotes.
    There may not be a space between the `<<` and the identifier,
    unless the identifier is explicitly quoted.  The terminating string
    must appear by itself (unquoted and with no surrounding whitespace)
    on the terminating line.

    If the terminating string is quoted, the type of quotes used determine
    the treatment of the text.

    - Double Quotes

        Double quotes indicate that the text will be interpolated using exactly
        the same rules as normal double quoted strings.

                  print <<EOF;
               The price is $Price.
               EOF
            
                  print << "EOF"; # same as above
               The price is $Price.
               EOF
            
            

    - Single Quotes

        Single quotes indicate the text is to be treated literally with no
        interpolation of its content.  This is similar to single quoted
        strings except that backslashes have no special meaning, with `\\`
        being treated as two backslashes and not one as they would in every
        other quoting construct.

        Just as in the shell, a backslashed bareword following the `<<`
        means the same thing as a single-quoted string does:

                   $cost = <<'VISTA';  # hasta la ...
               That'll be $10 please, ma'am.
               VISTA
            
                   $cost = <<\VISTA;   # Same thing!
               That'll be $10 please, ma'am.
               VISTA
            

        This is the only form of quoting in perl where there is no need
        to worry about escaping content, something that code generators
        can and do make good use of.

    - Backticks

        The content of the here doc is treated just as it would be if the
        string were embedded in backticks.  Thus the content is interpolated
        as though it were double quoted and then executed via the shell, with
        the results of the execution returned.

                  print << `EOC`; # execute command and get results
               echo hi there
               EOC
            

    - Indented Here-docs

        The here-doc modifier `~` allows you to indent your here-docs to make
        the code more readable:

               if ($some_var) {
                 print <<~EOF;
                   This is a here-doc
                   EOF
               }
            

        This will print...

               This is a here-doc
            

        ...with no leading whitespace.

        The delimiter is used to determine the **exact** whitespace to
        remove from the beginning of each line.  All lines **must** have
        at least the same starting whitespace (except lines only
        containing a newline) or perl will croak.  Tabs and spaces can
        be mixed, but are matched exactly.  One tab will not be equal to
        8 spaces!

        Additional beginning whitespace (beyond what preceded the
        delimiter) will be preserved:

               print <<~EOF;
                 This text is not indented
                   This text is indented with two spaces
                           This text is indented with two tabs
                 EOF
            

        Finally, the modifier may be used with all of the forms
        mentioned above:

               <<~\EOF;
               <<~'EOF'
               <<~"EOF"
               <<~`EOF`
            

        And whitespace may be used between the `~` and quoted delimiters:

               <<~ 'EOF'; # ... "EOF", `EOF`
            

    It is possible to stack multiple here-docs in a row:

              print <<"foo", <<"bar"; # you can stack them
           I said foo.
           foo
           I said bar.
           bar
        
              myfunc(<< "THIS", 23, <<'THAT');
           Here's a line
           or two.
           THIS
           and here's another.
           THAT
        

    Just don't forget that you have to put a semicolon on the end
    to finish the statement, as Perl doesn't know you're not going to
    try to do this:

              print <<ABC
           179231
           ABC
              + 20;
        

    If you want to remove the line terminator from your here-docs,
    use `chomp()`.

           chomp($string = <<'END');
           This is a string.
           END
        

    If you want your here-docs to be indented with the rest of the code,
    use the `<<~FOO` construct described under ["Indented Here-docs"](#indented-here-docs):

           $quote = <<~'FINIS';
              The Road goes ever on and on,
              down from the door where it began.
              FINIS
        

    If you use a here-doc within a delimited construct, such as in `s///eg`,
    the quoted material must still come on the line following the
    `<<FOO` marker, which means it may be inside the delimited
    construct:

           s/this/<<E . 'that'
           the other
           E
            . 'more '/eg;
        

    It works this way as of Perl 5.18.  Historically, it was inconsistent, and
    you would have to write

           s/this/<<E . 'that'
            . 'more '/eg;
           the other
           E
        

    outside of string evals.

    Additionally, quoting rules for the end-of-string identifier are
    unrelated to Perl's quoting rules.  `q()`, `qq()`, and the like are not
    supported in place of `''` and `""`, and the only interpolation is for
    backslashing the quoting character:

           print << "abc\"def";
           testing...
           abc"def
        

    Finally, quoted strings cannot span multiple lines.  The general rule is
    that the identifier must be a string literal.  Stick with that, and you
    should be safe.

## Gory details of parsing quoted constructs


When presented with something that might have several different
interpretations, Perl uses the **DWIM** (that's "Do What I Mean")
principle to pick the most probable interpretation.  This strategy
is so successful that Perl programmers often do not suspect the
ambivalence of what they write.  But from time to time, Perl's
notions differ substantially from what the author honestly meant.

This section hopes to clarify how Perl handles quoted constructs.
Although the most common reason to learn this is to unravel labyrinthine
regular expressions, because the initial steps of parsing are the
same for all quoting operators, they are all discussed together.

The most important Perl parsing rule is the first one discussed
below: when processing a quoted construct, Perl first finds the end
of that construct, then interprets its contents.  If you understand
this rule, you may skip the rest of this section on the first
reading.  The other rules are likely to contradict the user's
expectations much less frequently than this first one.

Some passes discussed below are performed concurrently, but because
their results are the same, we consider them individually.  For different
quoting constructs, Perl performs different numbers of passes, from
one to four, but these passes are always performed in the same order.

- Finding the end

    The first pass is finding the end of the quoted construct.  This results
    in saving to a safe location a copy of the text (between the starting
    and ending delimiters), normalized as necessary to avoid needing to know
    what the original delimiters were.

    If the construct is a here-doc, the ending delimiter is a line
    that has a terminating string as the content.  Therefore `<<EOF` is
    terminated by `EOF` immediately followed by `"\n"` and starting
    from the first column of the terminating line.
    When searching for the terminating line of a here-doc, nothing
    is skipped.  In other words, lines after the here-doc syntax
    are compared with the terminating string line by line.

    For the constructs except here-docs, single characters are used as starting
    and ending delimiters.  If the starting delimiter is an opening punctuation
    (that is `(`, `[`, `{`, or `<`), the ending delimiter is the
    corresponding closing punctuation (that is `)`, `]`, `}`, or `>`).
    If the starting delimiter is an unpaired character like `/` or a closing
    punctuation, the ending delimiter is the same as the starting delimiter.
    Therefore a `/` terminates a `qq//` construct, while a `]` terminates
    both `qq[]` and `qq]]` constructs.

    When searching for single-character delimiters, escaped delimiters
    and `\\` are skipped.  For example, while searching for terminating `/`,
    combinations of `\\` and `\/` are skipped.  If the delimiters are
    bracketing, nested pairs are also skipped.  For example, while searching
    for a closing `]` paired with the opening `[`, combinations of `\\`, `\]`,
    and `\[` are all skipped, and nested `[` and `]` are skipped as well.
    However, when backslashes are used as the delimiters (like `qq\\` and
    `tr\\\`), nothing is skipped.
    During the search for the end, backslashes that escape delimiters or
    other backslashes are removed (exactly speaking, they are not copied to the
    safe location).

    For constructs with three-part delimiters (`s///`, `y///`, and
    `tr///`), the search is repeated once more.
    If the first delimiter is not an opening punctuation, the three delimiters must
    be the same, such as `s!!!` and `tr)))`,
    in which case the second delimiter
    terminates the left part and starts the right part at once.
    If the left part is delimited by bracketing punctuation (that is `()`,
    `[]`, `{}`, or `<>`), the right part needs another pair of
    delimiters such as `s(){}` and `tr[]//`.  In these cases, whitespace
    and comments are allowed between the two parts, although the comment must follow
    at least one whitespace character; otherwise a character expected as the
    start of the comment may be regarded as the starting delimiter of the right part.

    During this search no attention is paid to the semantics of the construct.
    Thus:

           "$hash{"$foo/$bar"}"
        

    or:

           m/
             bar       # NOT a comment, this slash / terminated m//!
            /x
        

    do not form legal quoted expressions.   The quoted part ends on the
    first `"` and `/`, and the rest happens to be a syntax error.
    Because the slash that terminated `m//` was followed by a `SPACE`,
    the example above is not `m//x`, but rather `m//` with no `/x`
    modifier.  So the embedded `#` is interpreted as a literal `#`.

    Also no attention is paid to `\c\` (multichar control char syntax) during
    this search.  Thus the second `\` in `qq/\c\/` is interpreted as a part
    of `\/`, and the following `/` is not recognized as a delimiter.
    Instead, use `\034` or `\x1c` at the end of quoted constructs.

- Interpolation


    The next step is interpolation in the text obtained, which is now
    delimiter-independent.  There are multiple cases.

    - `<<'EOF'`

        No interpolation is performed.
        Note that the combination `\\` is left intact, since escaped delimiters
        are not available for here-docs.

    - `m''`, the pattern of `s'''`

        No interpolation is performed at this stage.
        Any backslashed sequences including `\\` are treated at the stage
        to ["parsing regular expressions"](#parsing-regular-expressions).

    - `''`, `q//`, `tr'''`, `y'''`, the replacement of `s'''`

        The only interpolation is removal of `\` from pairs of `\\`.
        Therefore `"-"` in `tr'''` and `y'''` is treated literally
        as a hyphen and no character range is available.
        `\1` in the replacement of `s'''` does not work as `$1`.

    - `tr///`, `y///`

        No variable interpolation occurs.  String modifying combinations for
        case and quoting such as `\Q`, `\U`, and `\E` are not recognized.
        The other escape sequences such as `\200` and `\t` and backslashed
        characters such as `\\` and `\-` are converted to appropriate literals.
        The character `"-"` is treated specially and therefore `\-` is treated
        as a literal `"-"`.

    - `""`, ``` `` ```, `qq//`, `qx//`, `<file*glob>`, `<<"EOF"`

        `\Q`, `\U`, `\u`, `\L`, `\l`, `\F` (possibly paired with `\E`) are
        converted to corresponding Perl constructs.  Thus, `"$foo\Qbaz$bar"`
        is converted to `$foo . (quotemeta("baz" . $bar))` internally.
        The other escape sequences such as `\200` and `\t` and backslashed
        characters such as `\\` and `\-` are replaced with appropriate
        expansions.

        Let it be stressed that _whatever falls between `\Q` and `\E`_
        is interpolated in the usual way.  Something like `"\Q\\E"` has
        no `\E` inside.  Instead, it has `\Q`, `\\`, and `E`, so the
        result is the same as for `"\\\\E"`.  As a general rule, backslashes
        between `\Q` and `\E` may lead to counterintuitive results.  So,
        `"\Q\t\E"` is converted to `quotemeta("\t")`, which is the same
        as `"\\\t"` (since TAB is not alphanumeric).  Note also that:

             $str = '\t';
             return "\Q$str";
            

        may be closer to the conjectural _intention_ of the writer of `"\Q\t\E"`.

        Interpolated scalars and arrays are converted internally to the `join` and
        `"."` catenation operations.  Thus, `"$foo XXX '@arr'"` becomes:

             $foo . " XXX '" . (join $", @arr) . "'";
            

        All operations above are performed simultaneously, left to right.

        Because the result of `"\Q _STRING_ \E"` has all metacharacters
        quoted, there is no way to insert a literal `$` or `@` inside a
        `\Q\E` pair.  If protected by `\`, `$` will be quoted to become
        `"\\\$"`; if not, it is interpreted as the start of an interpolated
        scalar.

        Note also that the interpolation code needs to make a decision on
        where the interpolated scalar ends.  For instance, whether
        `"a $x -> {c}"` really means:

             "a " . $x . " -> {c}";
            

        or:

             "a " . $x -> {c};
            

        Most of the time, the longest possible text that does not include
        spaces between components and which contains matching braces or
        brackets.  because the outcome may be determined by voting based
        on heuristic estimators, the result is not strictly predictable.
        Fortunately, it's usually correct for ambiguous cases.

    - the replacement of `s///`

        Processing of `\Q`, `\U`, `\u`, `\L`, `\l`, `\F` and interpolation
        happens as with `qq//` constructs.

        It is at this step that `\1` is begrudgingly converted to `$1` in
        the replacement text of `s///`, in order to correct the incorrigible
        _sed_ hackers who haven't picked up the saner idiom yet.  A warning
        is emitted if the `use warnings` pragma or the **-w** command-line flag
        (that is, the `$^W` variable) was set.

    - `RE` in `m?RE?`, `/RE/`, `m/RE/`, `s/RE/foo/`,

        Processing of `\Q`, `\U`, `\u`, `\L`, `\l`, `\F`, `\E`,
        and interpolation happens (almost) as with `qq//` constructs.

        Processing of `\N{...}` is also done here, and compiled into an intermediate
        form for the regex compiler.  (This is because, as mentioned below, the regex
        compilation may be done at execution time, and `\N{...}` is a compile-time
        construct.)

        However any other combinations of `\` followed by a character
        are not substituted but only skipped, in order to parse them
        as regular expressions at the following step.
        As `\c` is skipped at this step, `@` of `\c@` in RE is possibly
        treated as an array symbol (for example `@foo`),
        even though the same text in `qq//` gives interpolation of `\c@`.

        Code blocks such as `(?{BLOCK})` are handled by temporarily passing control
        back to the perl parser, in a similar way that an interpolated array
        subscript expression such as `"foo$array[1+f("[xyz")]bar"` would be.

        Moreover, inside `(?{BLOCK})`, `(?# comment )`, and
        a `#`-comment in a `/x`-regular expression, no processing is
        performed whatsoever.  This is the first step at which the presence
        of the `/x` modifier is relevant.

        Interpolation in patterns has several quirks: `$|`, `$(`, `$)`, `@+`
        and `@-` are not interpolated, and constructs `$var[SOMETHING]` are
        voted (by several different estimators) to be either an array element
        or `$var` followed by an RE alternative.  This is where the notation
        `${arr[$bar]}` comes handy: `/${arr[0-9]}/` is interpreted as
        array element `-9`, not as a regular expression from the variable
        `$arr` followed by a digit, which would be the interpretation of
        `/$arr[0-9]/`.  Since voting among different estimators may occur,
        the result is not predictable.

        The lack of processing of `\\` creates specific restrictions on
        the post-processed text.  If the delimiter is `/`, one cannot get
        the combination `\/` into the result of this step.  `/` will
        finish the regular expression, `\/` will be stripped to `/` on
        the previous step, and `\\/` will be left as is.  Because `/` is
        equivalent to `\/` inside a regular expression, this does not
        matter unless the delimiter happens to be character special to the
        RE engine, such as in `s*foo*bar*`, `m[foo]`, or `m?foo?`; or an
        alphanumeric char, as in:

             m m ^ a \s* b mmx;
            

        In the RE above, which is intentionally obfuscated for illustration, the
        delimiter is `m`, the modifier is `mx`, and after delimiter-removal the
        RE is the same as for `m/ ^ a \s* b /mx`.  There's more than one
        reason you're encouraged to restrict your delimiters to non-alphanumeric,
        non-whitespace choices.

    This step is the last one for all constructs except regular expressions,
    which are processed further.

- parsing regular expressions


    Previous steps were performed during the compilation of Perl code,
    but this one happens at run time, although it may be optimized to
    be calculated at compile time if appropriate.  After preprocessing
    described above, and possibly after evaluation if concatenation,
    joining, casing translation, or metaquoting are involved, the
    resulting _string_ is passed to the RE engine for compilation.

    Whatever happens in the RE engine might be better discussed in [perlre](https://metacpan.org/pod/perlre),
    but for the sake of continuity, we shall do so here.

    This is another step where the presence of the `/x` modifier is
    relevant.  The RE engine scans the string from left to right and
    converts it into a finite automaton.

    Backslashed characters are either replaced with corresponding
    literal strings (as with `\{`), or else they generate special nodes
    in the finite automaton (as with `\b`).  Characters special to the
    RE engine (such as `|`) generate corresponding nodes or groups of
    nodes.  `(?#...)` comments are ignored.  All the rest is either
    converted to literal strings to match, or else is ignored (as is
    whitespace and `#`-style comments if `/x` is present).

    Parsing of the bracketed character class construct, `[...]`, is
    rather different than the rule used for the rest of the pattern.
    The terminator of this construct is found using the same rules as
    for finding the terminator of a `{}`-delimited construct, the only
    exception being that `]` immediately following `[` is treated as
    though preceded by a backslash.

    The terminator of runtime `(?{...})` is found by temporarily switching
    control to the perl parser, which should stop at the point where the
    logically balancing terminating `}` is found.

    It is possible to inspect both the string given to RE engine and the
    resulting finite automaton.  See the arguments `debug`/`debugcolor`
    in the `use [re](https://metacpan.org/pod/re)` pragma, as well as Perl's **-Dr** command-line
    switch documented in ["Command Switches" in perlrun](https://metacpan.org/pod/perlrun#Command-Switches).

- Optimization of regular expressions


    This step is listed for completeness only.  Since it does not change
    semantics, details of this step are not documented and are subject
    to change without notice.  This step is performed over the finite
    automaton that was generated during the previous pass.

    It is at this stage that `split()` silently optimizes `/^/` to
    mean `/^/m`.
