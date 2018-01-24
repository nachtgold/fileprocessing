# File processing

Sometimes I have to work with many files or files which contain a lot data. Most of the time they are hidden in complex storages like databases, but if not it is very painful.

That is why I did some experiments. What kind of script can handle single files with hundreds of gigabytes as well as millions of files?

As sample data I used a dump of [wikidata.org][1]. In my case I was interested in the [Turtle-format file][2], because it is a single file with ~30GB.

With the data I asked myself: what do I want to achieve? 

1. split the single file in separate files, one for each wikidata entity (~40 mio.)
2. organize the data in folders, so that standard operating systems do not have to handle millions of files in a single directory
 
My experience is, that the more common Linux programs have problems with more than 500000 files and Windows starts to struggle with more than 1 mio. files (large footprints, "argument list too long", no responses and so on ...)

I found really good tools for both steps. The program [csplit][3] can split a file based on its content. So I defined a regular expression which isolates the entities.

Luckilly the content of the dump is somewhat ordered and each definition of an entity starts with a line like `wdata:Q10003946 a schema:Dataset ;`. Every `wdata` should be a separator between two files. As second argument I used `{*}` because the amount of entities is unknown.

```shell
csplit '/wdata.*/' {*}`
```

* Footprint: ~5MB
* Duration: 3 days on a i5-6600@3.3GHz

The second step is organizing the data in a way, a typical operating system can work with it. Wikipedia uses a smart way to manage their images. They calculate a hash for each filename and use some of the first characters as foldername. The longer the hash, the fewer files will be grouped.

```
tree
├───latest
├───out
│   ├───0
│   │   ├───00
│   │ ...
│   ├───0f
│   │   ├───0f
...
```

The sample code moves all files from `latest` directory (where csplit is working in) into the structured folder under `out`.

```python
import hashlib
import os
from shutil import move
import sys

source_folder = 'latest'
output_folder = 'out'

if not os.path.exists(output_folder):
    os.mkdir(output_folder)

with os.scandir(source_folder) as it:
    for entry in it:
        if entry.is_file():
            hash_fn = hashlib.md5(entry.name.encode('utf-8')).hexdigest()
            
            out_folder_level_one = os.path.join(output_folder, hash_fn[:1])
            if not os.path.exists(out_folder_level_one):
                os.mkdir(out_folder_level_one)
            
            out_folder_level_two = os.path.join(output_folder, hash_fn[:1], hash_fn[:2])
            if not os.path.exists(out_folder_level_two):
                os.mkdir(out_folder_level_two)
            
            move(entry.path, os.path.join(out_folder_level_two, entry.name))
```

Finding a function, which can loop over a huge amount of files, was not that easy. I used to use `os.listdir`, but it freezes with more than 500000 files. That led me to `os.scandir`. That magical function nearly ignores the size of a folder and just loops the files. Perfect!

[1]: https://www.wikidata.org
[2]: https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.ttl.gz
[3]: https://www.gnu.org/software/coreutils/manual/html_node/csplit-invocation.html
