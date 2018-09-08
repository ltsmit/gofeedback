# gofeedback
Go 2 draft designs feedback

Let us first start to distinguish between different use of errors:

1.	Reporting an error to a predefined stream, so that it is known that something is wrong and we can start debugging. In this case, it would be nice to have all relevant information with minimal effort, so that bug hunting can be done quickly.
2.	Perform a specific action in the program, based on the error code.

The first case, is the classical printf(“This is the functionname, Relevant parameters: param1 = %v, param2 = %v, param3=%v\n”, param1, param2, param3).
The most basic form of debugging. Million programmers have written billions of printf’s to achieve this.

Why not automate this?
If err != nil, then we want to know as much information as possible, at least:
-	The source file
-	The linenumber
-	The function name
-	The calling parameters
-	The returned error

So, if r, err := os.Open(src) fails, we would like a kind of output of:
> test.go:30, func=os.Open, src=”input.txt”, Err=”File does not exist”

The Go toolchain can generate the code for this, if the function has a flag that this is desired. Similar to the convention that starting a function with an uppercase means ‘export’.
For now, let us assume that if we add an exclamation mark to the end of the function, the Go toolchain will generate code that will output the desired information.
So, the new code will become r, _ := os.Open!(src)

A few remarks:
-	The generated code to report is not always desired. For example, if the function can fail frequently, error reporting can flood the error output stream. Consider for example writing a lot of rows to a remote database that cannot be reached. Therefore, it is necessary that the programmer explicitly adds the desire that the Go toolchain adds debug reporting code, in case that err != nil.
-	Generating code for reporting parameters of a basic type is trivial. What about more complex types? You do not want that slice of a million elements printed. A clear documented (arbitrary) decision has to be made here. For example, print the length of the slice, the first x elements, then … and then the last x elements. In most situations this will do. If this is not ok for the situation, then a programmer can still write their own err reporting code. Similar decisions have to be made for other complex types. Mostly they will do the job, and otherwise you have to do the job yourself, as is always the case till now.
-	The output has to go somewhere. Default stderr, but it should be possible to redirect the stream. For example, to a logfile. This can be set once in the beginning of the program.
-	Probably, the Go toolchain should generate code for 1) reporting the error and 2) a return from the function. Not sure about this.
-	The go compiler should complain when an ! is added to a function call, which does not return an error.

Advantages of this approach:
-	No more writing of trivial debug printf like code
-	No more cluttering of the code with these non-essential lines
-	Really fast asking for standard debugging info, just say !


The second case (perform specific action based on error code), can be done with the existing construction:
If err != nil {
	Do a specific action
}

It does not help to move this code to a ‘generic’ handler on a different place in the program, because this code is not generic, but specific for this error. Moving it to a handler on a different location makes it more difficult to understand the program.

A rewrite of the CopyFile example of the draft design to this proposal:

```go
func CopyFile(src, dst string) error {

	r, _ := os.Open!(src)
	defer r.Close!()

	w, _ := os.Create!(dst)

	err := io.Copy(w, r)
	if err!= nil {
		w.Close() // no ! added, because of return in generated code
		os.Remove(dst)
	}

	w.Close!()
	return nil
}
```
