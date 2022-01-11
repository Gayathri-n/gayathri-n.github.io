---
layout: post
author: Gayathri
tag: Javascript
---

My friend's server was infected with a Ransomware because of which all his files were encrypted and had an extension `.eking`. Not much was there to do but thankfully he had a backup of his files which could be restored. His server was formatted and the files were restored from the backup. But a few folders had just one file (`Thumbs.db`) that was encryted with `eking` and since his folders were nested, it would be a tedious process to manually check the folders and delete that specific file(s). Instead I offered him a NodeJS script that he can run on his Windows server that will delete that `.eking` file wherever that is present.

```js
const fs = require('fs');
var path = require('path')
const directory = "D:\\myDirectory\\"

function deleteFromDir(directory) {
    fs.readdir(directory, (err, files) => {
        files.forEach(file => {
            if (fs.statSync(path.join(directory, file)).isDirectory()) {
                deleteFromDir(path.join(directory, file))
            }
            if (path.extname(file) == '.eking') fs.unlink(path.join(directory, file), (err => {
                if (err) console.log(err);
                else {
                    console.log("\n Deleted file:" + file);
                }
            }));
        });
    });
}

deleteFromDir(directory);
```

## Learnings

- For windows, `\\` is used as the path separator.
- If you scan a whole drive like `D:`, there maybe some folders like `System Volume Information` that cannot be opened due to permission issues. It was difficult to exclude such folders so I resorted to run this sub folder-wise.
- I recommend using `path.join` and not string concatenation as it takes care of the platform specific details like path separator.



