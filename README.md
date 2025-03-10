# linkify-it-py

[![CI](https://github.com/tsutsu3/linkify-it-py/workflows/CI/badge.svg?branch=main)](https://github.com/tsutsu3/linkify-it-py/actions)
[![pypi](https://img.shields.io/pypi/v/linkify-it-py)](https://pypi.org/project/linkify-it-py/)
[![Anaconda-Server Badge](https://anaconda.org/conda-forge/linkify-it-py/badges/version.svg)](https://anaconda.org/conda-forge/linkify-it-py)
[![codecov](https://codecov.io/gh/tsutsu3/linkify-it-py/branch/main/graph/badge.svg)](https://codecov.io/gh/tsutsu3/linkify-it-py)
[![Maintainability](https://api.codeclimate.com/v1/badges/6341fd3ec5f05fde392f/maintainability)](https://codeclimate.com/github/tsutsu3/linkify-it-py/maintainability)
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Ftsutsu3%2Flinkify-it-py.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Ftsutsu3%2Flinkify-it-py?ref=badge_shield)

This is Python port of [linkify-it](https://github.com/markdown-it/linkify-it).

> Links recognition library with FULL unicode support.
> Focused on high quality link patterns detection in plain text.

__[Javascript Demo](http://markdown-it.github.io/linkify-it/)__

Why it's awesome:

- Full unicode support, _with astral characters_!
- International domains support.
- Allows rules extension & custom normalizers.


## Install

```bash
pip install linkify-it-py
```

or

```bash
conda install -c conda-forge linkify-it-py
```

## Usage examples

### Example 1. Simple use

```python
from linkify_it import LinkifyIt


linkify = LinkifyIt()

print(linkify.test("Site github.com!"))
# => True

print(linkify.match("Site github.com!"))
# => [linkify_it.main.Match({
#         'schema': '',
#         'index': 5,
#         'last_index': 15,
#         'raw': 'github.com',
#         'text': 'github.com',
#         'url': 'http://github.com'
#     }]
```

### Example 2. With options

```python
from linkify_it import LinkifyIt
from linkify_it.tlds import TLDS


# Reload full tlds list & add unofficial `.onion` domain.
linkify = (
    LinkifyIt()
    .tlds(TLDS)               # Reload with full tlds list
    .tlds("onion", True)      # Add unofficial `.onion` domain
    .add("git:", "http:")     # Add `git:` protocol as "alias"
    .add("ftp:", None)        # Disable `ftp:` protocol
    .set({"fuzzy_ip": True})  # Enable IPs in fuzzy links (without schema)
)
print(linkify.test("Site tamanegi.onion!"))
# => True

print(linkify.match("Site tamanegi.onion!"))
# => [linkify_it.main.Match({
#         'schema': '',
#         'index': 5,
#         'last_index': 19,
#         'raw': 'tamanegi.onion',
#         'text': 'tamanegi.onion',
#         'url': 'http://tamanegi.onion'
#     }]
```

### Example 3. Add twitter mentions handler

```python
from linkify_it import LinkifyIt


linkify = LinkifyIt()

def validate(obj, text, pos):
    tail = text[pos:]

    if not obj.re.get("twitter"):
        obj.re["twitter"] = re.compile(
            "^([a-zA-Z0-9_]){1,15}(?!_)(?=$|" + obj.re["src_ZPCc"] + ")"
        )
    if obj.re["twitter"].search(tail):
        if pos > 2 and tail[pos - 2] == "@":
            return False
        return len(obj.re["twitter"].search(tail).group())
    return 0

def normalize(obj, match):
    match.url = "https://twitter.com/" + re.sub(r"^@", "", match.url)

linkify.add("@", {"validate": validate, "normalize": normalize})
```


## API

[API documentation](https://linkify-it-py.readthedocs.io/en/latest/)

### LinkifyIt(schemas, options)

Creates new linkifier instance with optional additional schemas.

By default understands:

- `http(s)://...` , `ftp://...`, `mailto:...` & `//...` links
- "fuzzy" links and emails (google.com, foo@bar.com).

`schemas` is an dict, where each key/value describes protocol/rule:

- __key__ - link prefix (usually, protocol name with `:` at the end, `skype:`
  for example). `linkify-it-py` makes sure that prefix is not preceded with
  alphanumeric char.
- __value__ - rule to check tail after link prefix
  - _str_
    - just alias to existing rule
  - _dict_
    - _validate_ - either a `re.Pattern` (start with `^`, and don't include the
      link prefix itself), or a validator `function` which, given arguments
      _self_, _text_ and _pos_, returns the length of a match in _text_
      starting at index _pos_.  _pos_ is the index right after the link prefix.
      _self_ can be used to access the linkify object to cache data.
    - _normalize_ - optional function to normalize text & url of matched result
      (for example, for twitter mentions).

`options`:

- __fuzzy_link__ - recognize URL-s without `http(s)://` head. Default `True`.
- __fuzzy_ip__ - allow IPs in fuzzy links above. Can conflict with some texts
  like version numbers. Default `False`.
- __fuzzy_email__ - recognize emails without `mailto:` prefix. Default `True`.
- __---__ - set `True` to terminate link with `---` (if it's considered as long dash).


### .test(text)

Searches linkifiable pattern and returns `True` on success or `False` on fail.


### .pretest(text)

Quick check if link MAY BE can exist. Can be used to optimize more expensive
`.test()` calls. Return `False` if link can not be found, `True` - if `.test()`
call needed to know exactly.


### .test_schema_at(text, name, position)

Similar to `.test()` but checks only specific protocol tail exactly at given
position. Returns length of found pattern (0 on fail).


### .match(text)

Returns `list` of found link matches or null if nothing found.

Each match has:

- __schema__ - link schema, can be empty for fuzzy links, or `//` for
  protocol-neutral links.
- __index__ - offset of matched text
- __last_index__ - index of next char after mathch end
- __raw__ - matched text
- __text__ - normalized text
- __url__ - link, generated from matched text

### .matchAtStart(text)

Checks if a match exists at the start of the string. Returns `Match`
(see docs for `match(text)`) or null if no URL is at the start.
Doesn't work with fuzzy links.

### .tlds(list_tlds, keep_old=False)

Load (or merge) new tlds list. Those are needed for fuzzy links (without schema)
to avoid false positives. By default:

- 2-letter root zones are ok.
- biz|com|edu|gov|net|org|pro|web|xxx|aero|asia|coop|info|museum|name|shop|рф are ok.
- encoded (`xn--...`) root zones are ok.

If that's not enough, you can reload defaults with more detailed zones list.

### .add(key, value)

Add a new schema to the schemas object. As described in the constructor
definition, `key` is a link prefix (`skype:`, for example), and `value`
is a `str` to alias to another schema, or an `dict` with `validate` and
optionally `normalize` definitions.  To disable an existing rule, use
`.add(key, None)`.


### .set(options)

Override default options. Missed properties will not be changed.


## License

[MIT](https://github.com/tsutsu3/linkify-it-py/blob/master/LICENSE)


[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Ftsutsu3%2Flinkify-it-py.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Ftsutsu3%2Flinkify-it-py?ref=badge_large)