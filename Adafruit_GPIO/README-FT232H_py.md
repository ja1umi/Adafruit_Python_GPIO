

About modification to FT232H.py
-------------------------

**inb4 tl;dr (in  before, too long; didn't read)**
In my belief, *FT232H.py*, which is included in *Adafruit Python GPIO library* is incompatible with *libtfdi1* version 1.3. and it needs modification to make it work. I made an ugly patch for this issue.

**Tell me more**
FT232H Breakout did not work with *Adafruit Python GPIO library* on my Raspberry pi 3 with *libftdi1-1.3* installed. The followings are the steps I did:

 1. Firstly, *libftdi* and its dependencies were installed as per [the Adafruit's tutorial](https://learn.adafruit.com/adafruit-ft232h-breakout/linux-setup "Linux SetUp") (However, I installed *libftdi1-1.**3*** (the latest version, as of December 23, 2016), instead of 1.2.
 2. To test the installation, I tried to load (import) the libraries manually (*Adafruit Python GPIO library* and *libftdi*) in this order after opening the Python interpreter.
 3. The libraries were successfully loaded with no error.
 4. I encountered an error, meaning that invalid number of parameters were passed to the function *ftdi_write_data()* after executing  [gpio_test.py script](https://learn.adafruit.com/adafruit-ft232h-breakout/gpio "script") (this script is shown in the tutorial).

After some investigation, I found the followings:

 - The *ftdi_write_data* function provided by *libftdi* takes 3 parameters; pointer to struct *ftdi_context*, pointer to the buffer containing data to be written and size of the buffer (reference 1).
 - *FT232H.py*, which is incluided in the Adaruit's library, tries to pass all these 3 parameters to *libftdi*.
 - I am quite new to SWIG, however, it seems that the interface file *python/ftdi1.i*, which is found in libftdi source distribution package, was configured to accept only first 2 parameters. See the codes excerpted from *libftdi1-1.3/python/ftdi1.i* below.


```
    %define ftdi_write_data_docstring
    "write_data(context, data) -> return_code"
    %enddef
    %feature("autodoc", ftdi_write_data_docstring) ftdi_write_data;
    %typemap(in,numinputs=1) (const unsigned char *buf, int size) %{ $1 = (unsigned char*)str2charp_size($input, &$2); %}
    int ftdi_write_data(struct ftdi_context *ftdi, const unsigned char *buf, int size);
    %clear (const unsigned char *buf, int size);`
```
So, I changed *%typemap* line in *ftdi1.i* shown above so that it can accept all 3 parameters. In particular, I increased the parameter for *numinputs* and then tried to rebuild the library, but in vain. My compilation interrupted with the message meaning numinputs > 1 is not supported. I have no idea why python cannot accept numinputs > 1.

I got stuck. Finally, I decided to tweak the code in *FT232H.py* so that it passes just 2 parameters (pointer to the structure and pointer to the buffer), like this:

```python
--- ./Adafruit_GPIO/FT232H.py	2016-12-23 01:15:52.482121109 +0000
+++ ./build/lib.linux-armv7l-2.7/Adafruit_GPIO/FT232H.py	2016-12-23 04:18:47.907020837 +0000
@@ -186,7 +186,8 @@
         #else:
         #	logger.debug('Modem status error {0}'.format(ret))
         length = len(string)
-        ret = ftdi.write_data(self._ctx, string, length)
+        #ret = ftdi.write_data(self._ctx, string, length)
+	ret = ftdi.write_data(self._ctx, string)
         # Log the string that was written in a python hex string format using a very
         # ugly one-liner list comprehension for brevity.
         #logger.debug('Wrote {0}'.format(''.join(['\\x{0:02X}'.format(ord(x)) for x in string])))
```

After tweaking the code, I re-installed the Adafruit's library.

I think that it is *ad hoc*, potentially insecure workaround but it worked (at least, for me).

**Reference**
[Defenition of function ftdi_write_data](https://www.intra2net.com/en/developer/libftdi/documentation/group__libftdi.html#ga01199788c36ba93352f155a79ea295e8 "link to ftdi_write_data() definition"), libftdi API documentation.

**Postscript:**
I noticed on January 6, 2017 that this issue was already [reported](https://github.com/adafruit/Adafruit_Python_GPIO/issues/50 "issues") last november and the way to solve this issue is to roll back libftdi to version 1.2.

> Written with [StackEdit](https://stackedit.io/).
