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
    

# BASICS

Most commonly used features every [Mojolicious](https://metacpan.org/pod/Mojolicious) developer should know about.

## Automatic rendering

The renderer can be manually started by calling the method
["render" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#render), but that's usually not necessary, because
it will get automatically called if nothing has been rendered after the router
finished its work. This also means you can have routes pointing only to
templates without actual actions.

     $c->render;
    

There is one big difference though, by calling it manually you can make sure
that templates use the current controller object, and not the default
controller specified with the attribute ["controller\_class" in Mojolicious](https://metacpan.org/pod/Mojolicious#controller_class).

     $c->render_later;
    

You can also disable automatic rendering with the method
["render\_later" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#render_later), which can be very useful to delay
rendering when a non-blocking operation has to be performed first.

## Rendering templates

The renderer will always try to detect the right template, but you can also use
the `template` stash value to render a specific one. Everything before the
last slash will be interpreted as the subdirectory path in which to find the
template.

     # foo/bar/baz.*.*
     $c->render(template => 'foo/bar/baz');
    

Choosing a specific `format` and `handler` is just as easy.

     # foo/bar/baz.txt.epl
     $c->render(template => 'foo/bar/baz', format => 'txt', handler => 'epl');
    

Because rendering a specific template is the most common task it also has a
shortcut.

     $c->render('foo/bar/baz');
    

If you're not sure in advance if a template actually exists, you can also use
the method ["render\_maybe" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#render_maybe) to try multiple
alternatives.

     $c->render_maybe('localized/baz') or $c->render('foo/bar/baz');
    

## Rendering to strings

Sometimes you might want to use the rendered result directly instead of
generating a response, for example, to send emails, this can be done with
["render\_to\_string" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#render_to_string).

     my $html = $c->render_to_string('mail');
    

No encoding will be performed, making it easy to reuse the result in other
templates or to generate binary data.

     my $pdf = $c->render_to_string('invoice', format => 'pdf');
     $c->render(data => $pdf, format => 'pdf');
    

All arguments passed will get localized automatically and are only available
during this render operation.

## Template variants

To make your application look great on many different devices you can also use
the `variant` stash value to choose between different variants of your
templates.

     # foo/bar/baz.html+phone.ep
     # foo/bar/baz.html.ep
     $c->render('foo/bar/baz', variant => 'phone');
    

This can be done very liberally since it only applies when a template with the
correct name actually exists and falls back to the generic one otherwise.

## Rendering inline templates

Some renderers such as `ep` allow templates to be passed `inline`.

     $c->render(inline => 'The result is <%= 1 + 1 %>.');
    

Since auto-detection depends on a path you might have to supply a `handler`
too.

     $c->render(inline => "<%= shift->param('foo') %>", handler => 'epl');
    

## Rendering text

Characters can be rendered to bytes with the `text` stash value, the given
content will be automatically encoded with ["encoding" in Mojolicious::Renderer](https://metacpan.org/pod/Mojolicious::Renderer#encoding).

     $c->render(text => 'I ♥ Mojolicious!');
    

## Rendering data

Bytes can be rendered with the `data` stash value, no encoding will be
performed.

     $c->render(data => $bytes);
    

## Rendering JSON

The `json` stash value allows you to pass Perl data structures to the renderer
which get directly encoded to JSON with [Mojo::JSON](https://metacpan.org/pod/Mojo::JSON).

     $c->render(json => {foo => [1, 'test', 3]});
    

## Status code

Response status codes can be changed with the `status` stash value.

     $c->render(text => 'Oops.', status => 500);
    

## Content type

The `Content-Type` header of the response is actually based on the MIME type
mapping of the `format` stash value.

     # Content-Type: text/plain
     $c->render(text => 'Hello.', format => 'txt');
    
     # Content-Type: image/png
     $c->render(data => $bytes, format => 'png');
    

These mappings can be easily extended or changed with ["types" in Mojolicious](https://metacpan.org/pod/Mojolicious#types).

     # Add new MIME type
     $app->types->type(md => 'text/markdown');
    

## Stash data

Any of the native Perl data types can be passed to templates as references
through the ["stash" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#stash).

     $c->stash(description => 'web framework');
     $c->stash(frameworks  => ['Catalyst', 'Mojolicious']);
     $c->stash(spinoffs    => {minion => 'job queue'});
    
     %= $description
     %= $frameworks->[1]
     %= $spinoffs->{minion}
    

Since everything is just Perl normal control structures just work.

     % for my $framework (@$frameworks) {
       <%= $framework %> is a <%= $description %>.
     % }
    
     % if (my $description = $spinoffs->{minion}) {
       Minion is a <%= $description %>.
     % }
    

For templates that might get rendered in different ways and where you're not
sure if a stash value will actually be set, you can just use the helper
["stash" in Mojolicious::Plugin::DefaultHelpers](https://metacpan.org/pod/Mojolicious::Plugin::DefaultHelpers#stash).

     % if (my $spinoffs = stash 'spinoffs') {
       Minion is a <%= $spinoffs->{minion} %>.
     % }
    

## Helpers

Helpers are little functions you can use in templates as well as application
and controller code.

     # Template
     %= dumper [1, 2, 3]
    
     # Application
     my $serialized = $app->dumper([1, 2, 3]);
    
     # Controller
     my $serialized = $c->dumper([1, 2, 3]);
    

We differentiate between default helpers, which are more general purpose like
["dumper" in Mojolicious::Plugin::DefaultHelpers](https://metacpan.org/pod/Mojolicious::Plugin::DefaultHelpers#dumper), and tag helpers like
["link\_to" in Mojolicious::Plugin::TagHelpers](https://metacpan.org/pod/Mojolicious::Plugin::TagHelpers#link_to), which are template specific and
mostly used to generate HTML tags.

     %= link_to Mojolicious => 'https://mojolicious.org'
    

In controllers you can also use the method ["helpers" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#helpers)
to fully qualify helper calls and ensure that they don't conflict with existing
methods you may already have.

     my $serialized = $c->helpers->dumper([1, 2, 3]);
    

A list of all built-in helpers can be found in
[Mojolicious::Plugin::DefaultHelpers](https://metacpan.org/pod/Mojolicious::Plugin::DefaultHelpers) and [Mojolicious::Plugin::TagHelpers](https://metacpan.org/pod/Mojolicious::Plugin::TagHelpers).

## Content negotiation

For resources with different representations and that require truly RESTful
content negotiation you can also use
["respond\_to" in Mojolicious::Plugin::DefaultHelpers](https://metacpan.org/pod/Mojolicious::Plugin::DefaultHelpers#respond_to) instead of
["render" in Mojolicious::Controller](https://metacpan.org/pod/Mojolicious::Controller#render).

     # /hello (Accept: application/json) -> "json"
     # /hello (Accept: application/xml)  -> "xml"
     # /hello.json                       -> "json"
     # /hello.xml                        -> "xml"
     # /hello?format=json                -> "json"
     # /hello?format=xml                 -> "xml"
     $c->respond_to(
       json => {json => {hello => 'world'}},
       xml  => {text => '<hello>world</hello>'}
     );
    

The best possible representation will be automatically selected from the
`format` `GET`/`POST` parameter, `format` stash value or `Accept` request
header and stored in the `format` stash value. To change MIME type mappings for
the `Accept` request header or the `Content-Type` response header you can use
["types" in Mojolicious](https://metacpan.org/pod/Mojolicious#types).

     $c->respond_to(
       json => {json => {hello => 'world'}},
       html => sub {
         $c->content_for(head => '<meta name="author" content="sri">');
         $c->render(template => 'hello', message => 'world')
       }
     );
    

Callbacks can be used for representations that are too complex to fit into a
single render call.

     # /hello (Accept: application/json) -> "json"
     # /hello (Accept: text/html)        -> "html"
     # /hello (Accept: image/png)        -> "any"
     # /hello.json                       -> "json"
     # /hello.html                       -> "html"
     # /hello.png                        -> "any"
     # /hello?format=json                -> "json"
     # /hello?format=html                -> "html"
     # /hello?format=png                 -> "any"
     $c->respond_to(
       json => {json => {hello => 'world'}},
       html => {template => 'hello', message => 'world'},
       any  => {text => '', status => 204}
     );
    

And if no viable representation could be found, the `any` fallback will be
used or an empty `204` response rendered automatically.

     # /hello                      -> "html"
     # /hello (Accept: text/html)  -> "html"
     # /hello (Accept: text/xml)   -> "xml"
     # /hello (Accept: text/plain) -> undef
     # /hello.html                 -> "html"
     # /hello.xml                  -> "xml"
     # /hello.txt                  -> undef
     # /hello?format=html          -> "html"
     # /hello?format=xml           -> "xml"
     # /hello?format=txt           -> undef
     if (my $format = $c->accepts('html', 'xml')) {
       ...
     }
    

For even more advanced negotiation logic you can also use the helper
["accepts" in Mojolicious::Plugin::DefaultHelpers](https://metacpan.org/pod/Mojolicious::Plugin::DefaultHelpers#accepts).

# POD ERRORS

Hey! **The above document had some coding errors, which are explained below:**

- Around line 61:

    Non-ASCII character seen before =encoding in '♥'. Assuming CP1252
