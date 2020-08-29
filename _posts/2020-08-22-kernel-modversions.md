---
layout: post
title: Defeating Kernel Modules Version Check
date: '2020-08-22T00:00:00.000+00:00'
tags:
- 
modified_time: '2020-08-22T00:00:00.000+00:00'
redirect_from: /2020/08/classic.html
---

For various reasons, it may be necessary to compile modules for an existing binary kernel.
If your module is rejected by the kernel with the `disagrees about version of symbol` error like:

~~~
[root@localhost /mnt]# insmod dm9000.ko
dm9000: disagrees about version of symbol platform_driver_register
dm9000: Unknown symbol platform_driver_register (err -22)
~~~

which means the binary kernel was compiled with CONFIG_MODVERSIONS and it has had some modifications causing its definitions of data structures, function prototypes, etc to diverge from the versions used when compiling  _your_  module.

It is likely that your module is badly incompatible and can't work. But it is also possible that the module
does not use the part of the kernel API that was changed. Or the changes do not actually cause any compatibility issues.
When computing a symbol "version", which is actually, a CRC32
of its extended signature, the kernel takes into account argument and structure member names and exact definitions of 
used data types. If say an identifier was renamed or a data type was changed from `u32` to `int`, this will also cause 
the module to break. 

In any case, to trick the kernel to load the module, the `__versions` section of the module can be updated with the versions (CRC32's) that the kernel expects. Ideally, you need a binary version of the module that the kernel does load.

Then, its  _correct_  symbol versions can be extracted with:

~~~
m=dm9000
arm-linux-gnueabi-objcopy $m.ko --dump-section __versions=$m.versions
~~~

It is possible that the number of entries in the version table has also changed. Therefore, it may not be a good idea to just replace the section. Instead, I use the following Python script `update_versions.py` to update the versions  _in-place_ :

{% highlight python %}
#!/usr/bin/env python

import sys

SIZEOF_MODVERSION_INFO=64
SIZEOF_CRC32=4

if len(sys.argv) <= 1:
	print "Need an argument: <model-file>"
	sys.exit(1)

model_file = open(sys.argv[1], "r")

model = {}
while True:
	x = model_file.read(SIZEOF_MODVERSION_INFO)
	if len(x) < SIZEOF_MODVERSION_INFO:
		break
	model[x[SIZEOF_CRC32:]] = x[:SIZEOF_CRC32]

while True:
	x = sys.stdin.read(SIZEOF_MODVERSION_INFO)
	if len(x) < SIZEOF_MODVERSION_INFO:
		break
	s = x[SIZEOF_CRC32:]
	sys.stdout.write(model[s] + s)
{% endhighlight %}

Finally, the module's `__versions` section is extracted and replaced using the following:

~~~
m=dm9000
arm-linux-gnueabi-objcopy $m.ko --dump-section __versions=/dev/stdout | ./update_versions.py >$m-new.versions
arm-linux-gnueabi-objcopy $m.ko --update-section __versions=$m-new.versions
~~~
