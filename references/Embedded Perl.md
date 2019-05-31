## Embedded Perl

[Mojolicious](https://metacpan.org/pod/Mojolicious) includes a minimalistic but very powerful template system out of
the box called Embedded Perl or `ep` for short. It is based on
[Mojo::Template](https://metacpan.org/pod/Mojo::Template) and allows the embedding of Perl code right into actual
content using a small set of special tags and line start characters. For all
templates [strict](https://metacpan.org/pod/strict), [warnings](https://metacpan.org/pod/warnings), [utf8](https://metacpan.org/pod/utf8) and Perl 5.10 [features](https://metacpan.org/pod/feature) are
automatically enabled.

     <% Perl code %>
     <%= Perl expression, replaced with XML escaped result %>
     <%== Perl expression, replaced with result %>
     <%# Comment, useful for debugging %>
     <%% Replaced with "<%", useful for generating templates %>
     % Perl code line, treated as "<% line =%>" (explained later)
     %= Perl expression line, treated as "<%= line %>"
     %== Perl expression line, treated as "<%== line %>"
     %# Comment line, useful for debugging
     %% Replaced with "%", useful for generating templates
    

Tags and lines work pretty much the same, but depending on context one will
usually look a bit better. Semicolons get automatically appended to all
expressions.

     <% my $i = 10; %>
     <ul>
       <% for my $j (1 .. $i) { %>
         <li>
           <%= $j %>
         </li>
       <% } %>
     </ul>
    
     % my $i = 10;
     <ul>
       % for my $j (1 .. $i) {
         <li>
           %= $j
         </li>
       % }
     </ul>
    

Aside from differences in whitespace handling, both examples generate similar
Perl code, a naive translation could look like this.

     my $output = '';
     my $i = 10;
     $output .= '<ul>';
     for my $j (1 .. $i) {
       $output .= '<li>';
       $output .= xml_escape scalar + $j;
       $output .= '</li>';
     }
     $output .= '</ul>';
     return $output;
    

An additional equal sign can be used to disable escaping of the characters
`<`, `>`, `&`, `'` and `"` in results from Perl expressions, which
is the default to prevent XSS attacks against your application.

     <%= 'I ♥ Mojolicious!' %>
     <%== '<p>I ♥ Mojolicious!</p>' %>
    

Only [Mojo::ByteStream](https://metacpan.org/pod/Mojo::ByteStream) objects are excluded from automatic escaping.

     <%= b('<p>I ♥ Mojolicious!</p>') %>
    

Whitespace characters around tags can be trimmed by adding an additional equal
sign to the end of a tag.

     <% for (1 .. 3) { %>
       <%= 'Trim all whitespace characters around this expression' =%>
     <% } %>
    

Newline characters can be escaped with a backslash.

     This is <%= 1 + 1 %> a\
     single line
    

And a backslash in front of a newline character can be escaped with another
backslash.

     This will <%= 1 + 1 %> result\\
     in multiple\\
     lines
    

A newline character gets appended automatically to every template, unless the
last character is a backslash. And empty lines at the end of a template are
ignored.

     There is <%= 1 + 1 %> no newline at the end here\
    

At the beginning of the template, stash values that don't have invalid
characters in their name get automatically initialized as normal variables, and
the controller object as both `$self` and `$c`.

     $c->stash(name => 'tester');
    
     Hello <%= $name %> from <%= $c->tx->remote_address %>.
    

A prefix like `myapp.*` is commonly used for stash values that you don't want
to expose in templates.

     $c->stash('myapp.name' => 'tester');
    

There are also many helper functions available, but more about that later.

    <%= dumper {foo => 'bar'} %>

