---
title: mdbook目录生成以及适应typora公式
date: 2020-04-17 02:39:16
tags:
---
<!-- wp:paragraph --> [mdbook](https://rust-lang.github.io/mdBook/) 是一个rust写的的在线图书生成工具。原本目录需要自己写，写了个小脚本来基于文件名自动生成。另外我平时使用typora来写markdown，内联公式的符号和mdbook不同，脚本里也做了替换。Show you the code。

```
import os
import re
mypath = './src/'

pattern = '\$[^$]+\$'
prog = re.compile(pattern, re.M)
def is_md(file):
    return file.split('.')[-1] == 'md'

def search_file(root):
    mds = []
    for path, subdirs, files in os.walk(root):
        for name in files:
            f = os.path.join(path, name)
            if is_md(f):
                mds.append(f)
    return mds

def build_item(title, link, rank):
    indent = '    ' * (rank - 1)
    return '{}- [{}]({})'.format(indent, title.strip('#\n ').split(' ')[-1], link.replace(mypath, ''))

def replace_inline_math(string):
    placeholder = '|-|'
    string = string.replace('$$', placeholder)
    mathes = prog.findall(string)
    for math in mathes:
        new_math = '\\\\({}\\\\)'.format(math.strip('$'))
        string = string.replace(math, new_math)
    string = string.replace(placeholder, '$$')
    return string

files = sorted(search_file(mypath), key=lambda x: '.'.join(x.split('/')[-1].split('.')[0:-1]).split('-')[0])
summary = []
for file in files:
    rank = len(file.split('/')[-1].split('.')[0:-1])
    result = ''
    with open(file, 'r') as f:
        title = f.readline()
        summary.append(build_item(title, file, rank))
        result = '{}\n{}'.format(title, replace_inline_math(f.read()))
        #print(replace_inline_math('$$adfad$$adasdfasdf$asdfasdf$'))
    with open(file, 'w') as f:
        f.write(result)

print('# SUMMARY')
print('')
for l in summary:
    print(l)
```