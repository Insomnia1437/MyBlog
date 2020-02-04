---
title: YAML introduction
date: 2020-02-02 23:19:18
categories:
  - "Software"
tags:
  - "YAML"
  - "English"
---

```python
# @Time    : 2019-02-02
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

# A Short Report on YAML

> YAML means **“YAML Ain't a Markup Language”**. It uses Python-style indentation to indicate nesting, and use `[]` for lists and `{}` for maps.
>
> Since the name is recursive, it is a better option to explain YAML by YAML.

```yaml
--- # Syntax
basic syntax:
  - whitespace indentation is used for denoteing structure and `tab` is not allowed
  - comments begin with the `#`
  - List members are denoted by leading hyphen (`-`) with one member per line
  - use key:value to denote array
  - case sensitive
Support data structure:
  - map (also called dictory or hashes or key:value pair)
  - array or list or sequence
  - scalars

# basic components
map:
  - map1: this is a map
  - map2: {name: hello, id: world}
list:
    - cat
    - dog
    - cow
    - [monkey, elephant]
scalars:
  null: ~
  boolean:
    - TRUE # True (yaml1.1), string "Yes" (yaml1.2)
    - FALSE
    - yes
    - no
    - Yes
    - YeS # will be recognized as a string
  float:
    - 3.14
    - 2.99e+8
  int:
    - 65536
    - 0b1010 # dec: 10
  srting:
    - apple
    - "another string"
    - '\n will be escaped'
    - "\n will not be escaped"
    - data: |
        There once was a tall man from Ealing
        Who got on a bus to Darjeeling
          It said on the door
          "Please don't sit on the floor"
        So he carefully sat on the ceiling
    - info: >
        this is
        an
        info

        \n
        continued
        info
date and time:
  # {date}t{time}+timezone
  - iso8601: 2001-12-25t01:23:45.54+09:00
  - date: &epoch 1970-01-01
  - epochtime: *epoch
explicit type:
  str_true: !!str true # string rather than boolean
  str_int: !!str 123 # string rather than int
anchor and refers:
  - anchor: &anchor001
      Manager: Alice
      Programer: Bob
      Teacher: Dylan
      Student: Ruby
  - refer: *anchor001
  - merger:
      <<: *anchor001
      Student: Python
```

Parse this YAML file by Python3.

```python
import yaml
import pprint

f = open("./test.yaml")
a = yaml.load(f)
print(type(a))
pprint.pprint(a)
```

This is the output. It seems there are some errors about the **highlight** color.
```python
<class 'dict'>
{'Support data structure': ['map (also called dictory or hashes or key:value '
                            'pair)',
                            'array or list or sequence',
                            'scalars'],
 'anchor and refers': [{'anchor': {'Manager': 'Alice',
                                   'Programer': 'Bob',
                                   'Student': 'Ruby',
                                   'Teacher': 'Dylan'}},
                       {'refer': {'Manager': 'Alice',
                                  'Programer': 'Bob',
                                  'Student': 'Ruby',
                                  'Teacher': 'Dylan'}},
                       {'merger': {'Manager': 'Alice',
                                   'Programer': 'Bob',
                                   'Student': 'Python',
                                   'Teacher': 'Dylan'}}],
 'basic syntax': ['whitespace indentation is used for denoteing structure and '
                  '`tab` is not allowed',
                  'comments begin with the `#`',
                  'List members are denoted by leading hyphen (`-`) with one '
                  'member per line',
                  'use key:value to denote array',
                  'case sensitive'],
 'date and time': [{'iso8601': datetime.datetime(2001, 12, 24, 16, 23, 45, 540000)},
                   {'date': datetime.date(1970, 1, 1)},
                   {'epochtime': datetime.date(1970, 1, 1)}],
 'explicit type': {'str_int': '123', 'str_true': 'true'},
 'list': ['cat', 'dog', 'cow', ['monkey', 'elephant']],
 'map': [{'map1': 'this is a map'}, {'map2': {'id': 'world', 'name': 'hello'}}],
 'scalars': {None: None,
             'boolean': [True, False, True, False, True, 'YeS'],
             'float': [3.14, 299000000.0],
             'int': [65536, 10],
             'srting': ['apple',
                        'another string',
                        '\\n will be escaped',
                        '\n will not be escaped',
                        {'data': 'There once was a tall man from Ealing\n'
                                 'Who got on a bus to Darjeeling\n'
                                 '  It said on the door\n'
                                 '  "Please don\'t sit on the floor"\n'
                                 'So he carefully sat on the ceiling\n'},
                        {'info': 'this is an info\n\\n continued info\n'}]}}
```