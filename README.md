# File processing

Somethimes I have to work with huge or many files. Normally they are hidden in storages or databases, but if not it's very painfull.

That's why I made an experiment. What kind of script can handle files with hundreds of gigabytes or millions of files?

As sample data I used a dump of [wikidata.org][1]. In my case I was interested in the [Turtle-format file][2], because it is a single file with ~30GB.

With the data I made a plan, what I want to archive?

1. split the single file in separate files, one for each wikipdata entity (~40 mio.)
2. organize the data in folders, so that normal operating systems have hot to handle millions of files in a single directory
 
My experiences are, that the more common Linux programs have problems with more than 500.000 files and Windows starts to struggle with more than 1 mio. files (large footprints, "argument list too long", no responses and so on ...)

I found really good tools for both steps. The program [csplit][3] can split a file based on its content. So I defined a regular expression which isolates the entities.

Luckilly the content of dump is somewhat ordered and each definition of an entity starts with a line like `wdata:Q10003946 a schema:Dataset ;`. I stated, that every `wdata` should be separator between two files. As second argument I used `{*}` because the amount of entities is unknown.

```shell
csplit '/wdata.*/' {*}`
```

* Footprint: ~5MB
* Durration: 3 days on a i5-6600@3.3GHz

The second step is organizing the data in a way, a typical operating system can work with. Wikipedia uses a smart way to manage their images. They calculate a hash for each filename and use some of the first characters as foldername. The longer the hash is, the fewer files will be grouped.

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

The sample code moves all files from `latest` directory (where cplit is working on) into the structured folder under `out`.

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

Finding a function, which can loop over huge amount of files, was not that easy. Typically I used `os.listdir`, but it freezes with more than 500.000 files. That led me to `os.scandir`. That magical function nearly ignores the size of a folder and just loops the files. Perfect!

[1]: https://www.wikidata.org
[2]: https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.ttl.gz
[3]: https://www.gnu.org/software/coreutils/manual/html_node/csplit-invocation.html
