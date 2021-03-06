# contributing

There are two main areas where showthedocs could improve: supporting a new
language and improving an existing parser/annotator. The UI itself doesn't work
very well on small/touch screens, so if someone cares about that, feel free to
send suggestions or pull requests.

If you'd like to hack on the site but have a hard time navigating the code
base, please don't hesitate to reach out to me either on github or privately
and I'll answer any questions (and add missing documentation).

## supporting a new language

Adding a language requires the following:

1. writing a parser (or using an existing one)
1. using that parser to annotate queries
1. importing the documentation (usually a small subset of it) and annotating it
   as well

[This commit](https://github.com/idank/showthedocs/commit/14bcc7219efb77ae9df2141d0986e8645a883453) that added support for gitconfig can be used for guidance on the
various pieces of the code that need to be modified.

### writing a parser

The parsing that showthedocs needs is generally a lot more shallow than the one
used by the target language tooling. Python has a vast array of existing
parsers, so check what's out there before starting one from scratch. If you do
write one, prefer to write a lenient parser, since the purpose of showthedocs
isn't to validate.

The parser should produce some form of an [AST](https://github.com/idank/showthedocs/blob/master/showdocs/parsers/ast.py) that can be easily traversed by
the annotator (more on that next). In essence this is similar to what most
syntax highlighters do, but for the results to be more meaningful than "this is
a keyword", or "that is a string", parsing needs to go deeper and provide
things like "this is a SELECT statement, these are the table names", etc.

SQL is parsed using [sqlparse](https://github.com/andialbrecht/sqlparse). Nginx
and gitconfigs are parsed with pyparsing (see `showdocs/parsers/`).

### annotating code

This is the definiton of an annotation:

    class Annotation(object):
        '''An annotation selects a range in the input, assigns it a group and
        a list of arbitrary class names. Annotations translate to HTML by
        wrapping the range in a <span> tag. The group appears as the value of
        a data-showdocs attribute, likewise for the class names.

        A group is an arbitrary string that identifies the selected range. A group
        is visualized in a special manner in the user interface, depending on the
        decoration applied to it. A group exists to connect a piece of the input to
        its documentation, which will somewhere have a tag with the same
        data-showdocs attribute.

        The list of class names is currently only used to apply decorations.
        A decoration controls the display of the annotation in the UI. The most
        common one is a back decoration, which changes the background color and
        supports things like connecting links when hovering the annotation.'''
        def __init__(self, start, end, group, classes):
            if end <= start:
                raise ValueError('end smaller than start')
            if not isinstance(classes, list):
                raise ValueError('classes needs to be a list')

            self.start = start
            self.end = end
            self.group = group
            self.classes = classes

The annotation process consists of creating annotations for the input by
traversing the AST produced by the parser. It is the responsibility of the
annotator to assign groups to certain parts of the input and apply
decorations. Lastly, the annotator needs to request a piece of
documentation to appear in the result of the query.

For example, if we consider the query:

    SELECT * FROM foo;

we might see these annotations:

- `SELECT`, group=select, decorate=back
- `*`, group=column, decorate=back
- `FROM`, group=from, decorate=back
- `foo`, group=table, decorate=back

### documentation

The final piece is bringing the documentation of the target language into
showthedocs. All we care about is HTML that has a bunch of tags with
`data-showdocs` and some decorations applied to those.

[devdocs](http://devdocs.io) is a documentation aggregator to a vast array of
languages. It scrapes the online copy and does a bunch of modifications to the
downloaded HTML. The end result is ideal for our purposes.

A [repository](https://github.com/idank/showthedocs/blob/master/showdocs/repos/common.py#L16)
is in charge of building documentation and running
[filters](https://github.com/idank/showthedocs/blob/master/showdocs/filters/common.py#L3)
that among others modify the output to include `data-showdocs` attributes. There are currently
two base repositories. `DevDocsRepository` is based on devdocs and simply
shells out to its tools. The other one is `ScrapedRepository`, which has
utility functions for downloading web pages.

Filters should be written to add `data-showdocs` in strategic places of the
built HTML. See the nginx [repo](https://github.com/idank/showthedocs/blob/master/showdocs/repos/nginx.py) and [filters](https://github.com/idank/showthedocs/blob/master/showdocs/filters/nginx.py) for examples.

[getdocs](https://github.com/idank/showthedocs/blob/master/getdocs) is used to build repositories, and puts the output files in
a directory under `external/`.  An annotator can then request to include a file
from this directory by calling `self.add`, e.g. a file at
`external/foo/bar.html` will show up in the result of a query if its annotator
calls `self.add('foo/bar.html')`.
