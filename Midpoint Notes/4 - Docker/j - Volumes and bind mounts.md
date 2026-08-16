![[Pasted image 20260730232812.png]]

Data will be stored to an external storage, so if MYSQL will be removed, data will still be available, and can be fetched whenever another instance of MYSQL will run.

Use:
- `docker run -v <destination dir>:<source dir> mysql`
- example: `docker run -v /opt/datadir:/var/lib/mysql mysql`

### `-v` vs `--mount` 

![[Pasted image 20260730233309.png]]

The main difference is that `-v` (or `--volume`) is the older, shorthand syntax that combines multiple mounting behaviors into one flag, while `--mount` is the modern, explicit, and preferred syntax for all Docker storage types. 

While `-v` infers what you want to do based on the format of your input, `--mount` forces you to explicitly define the details, making it less error-prone and much easier to read. 

|Feature|`-v` / `--volume`|`--mount`|
|---|---|---|
|**Syntax Style**|Compact, shorthand string (`:` separated)|Key-value pairs (`=`, separated by commas)|
|**Clarity**|Ambiguous; format dictates behavior|Explicit; you must declare the `type`|
|**Missing Host Paths**|**Automatically creates** an empty directory on the host|**Throws an error** and fails to start the container|
|**Docker Swarm**|Not supported for standalone services|Fully supported and required|
|**Advanced Options**|Limited configuration capabilities|Supports all advanced storage options and volume driver tweaks|