# Installable Client Driver

The canonical ICD loader made public by Khronos does the following:

##### Linux

It opens the files `/etc/OpenCL/vendors/*.icd` which should all be text files with a single line, holding a shared library name. All such `*.icd` files will correspond to one platform in the application. The ICD will try to load these libraries via the C runtime's `dlopen()`, hence the system linker must be able to find them through means of the Linux distribution at hand (`ld.so.conf`, `LD_LIBRARY_PATH`, etc.).

##### Windows

```powershell
foreach ($Platform in (Get-Item HKLM:\SOFTWARE\Khronos\OpenCL\Vendors\)) { $Platform }
```

##### MacOS
