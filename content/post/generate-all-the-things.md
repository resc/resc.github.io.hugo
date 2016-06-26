+++
bigimg = ""
date = "2016-06-25T19:03:05+02:00"
subtitle = "or how to save yourself some work in the long run"
title = "Generate All The Things"
tags = ["C#", "Generators", "Reflection" ]

+++
Coding is a lot of work, and sometimes it gets repetitive too. I don't like repetitive work so most of my projects grow some kind of code generator.
Usually it starts with a unit test that outputs a simple class definition, and before you know it, it generates a complete data-access layer with [NHibernate](http://nhibernate.info/) mappings in [FluentNHibernate](http://www.fluentnhibernate.org/) that generate SQL create scripts... but that's a story for another post.

So, here's my default 'plan' for building ad-hoc code generators:

* Create a test project
* Add a [InternalsVisibleTo](https://msdn.microsoft.com/en-us/library/system.runtime.compilerservices.internalsvisibletoattribute(v=vs.110).aspx) attribute to the target project with the assembly name of the test project, so that the test project can access methods and properties defined as internal in the target project.
```csharp
// expose internals to the test project.
[assembly: InternalsVisibleTo("GenerateAllTheThings.Tests", AllInternalsVisible = true)]
```

* Create a class ```Source``` to get the location of the source code using the [CallerFilePath](https://msdn.microsoft.com/en-us/library/system.runtime.compilerservices.callerfilepathattribute(v=vs.110).aspx) attribute. 
```csharp
    /// <summary>
    /// Source contains methods to find the source file 
    /// and directory at the time of compilation.
    /// </summary>
    public static class Source
    {
        /// <summary> 
        /// Returns the source file path of the method that calls FilePath, 
        /// do not supply any arguments, the compiler will do that for you. 
        /// </summary>
        public static string FilePath([CallerFilePath] string callerFilePath = null)
        {
            return callerFilePath;
        }

        /// <summary> 
        /// returns the directory of the source file
        /// of the method that calls Directory.
        /// </summary>
        public static string Directory([CallerFilePath] string callerFilePath = null)
        {
            return System.IO.Path.GetDirectoryName(callerFilePath);
        }
    }
```
* put an internal static property ```SourceDirectory``` on any class in the target directory for the code generator like this: 
```csharp
    internal static string SourceDirectory
    {
        get { return Source.Directory(); }
    }
```
* Create at least two types that you want to generate code for immediately, this prevents overspecialization of the generator.
* Add a test class named ```Generator``` with a unit test called ```Generate``` that generates some code for classes using the [reflection api](https://msdn.microsoft.com/en-us/library/mt656691.aspx).
* Don't be afraid, embrace the [Inception effect](http://www.imdb.com/title/tt1375666/) (a.k.a. [recursion](https://en.wikipedia.org/wiki/Recursion_(computer_science))), generate types that the generator uses to generate more types. But stop if your brain starts hurting, it's a sign you've taken this too far. 
* Think about the conventions and project folder structure you want to use. Code generators are good tools to promote uniformity in coding and naming conventions, but those conventions have to be in place before they can be leveraged.  
* Add the generated files to the project, the [partial](https://msdn.microsoft.com/en-us/library/wa80x488.aspx) keyword is your friend if you want to attach generated code to an existing class.
* If the generator generates gibberish, correct the code in the generated file, and copy-paste the working code back into the generator and run the ```Generate``` test.
* Repeat the previous step as many times as needed.

As an example I've created a codegenerator that generates constructors and a simple [IXmlSerializable](https://msdn.microsoft.com/en-us/library/system.xml.serialization.ixmlserializable(v=vs.110).aspx) implementation for immutable message classes. [See the full code here](https://github.com/resc/wpfmagic/tree/master/GenerateAllTheThings)

This CodeWriter class helps keeping track of blocks and code indentation. I use it to write out code that reads as if it's hand coded, because I like it that way.
```csharp
 /// <summary> CodeWriter keeps track of indentation and blocks, to simplify writing code. </summary>
    public class CodeWriter : TextWriter
    {
        /// <summary> Creates a new file, or truncates an existing file </summary>
        public static CodeWriter CreateFile(string filepath)
        {
            return new CodeWriter(File.Open(filepath, FileMode.Create, FileAccess.Write, FileShare.Read));
        }

        private readonly TextWriter _w;
        private char _newline;
        private char[] _ignored
            ;
        private bool _writeIndent;
        private int _indentLevel;

        public CodeWriter() : this(new StringWriter())
        {
        }

        public CodeWriter(Stream s) : this(new StreamWriter(s))
        {
        }

        public CodeWriter(TextWriter w)
        {
            _w = w;
            SetNewLine(w.NewLine);
            IndentText = "    ";
            BlockStart = "{";
            BlockEnd = "}";
            BlockCommentStart = "/*";
            BlockCommentEnd = "*/";
            CommentStart = "//";
            CommentEnd = w.NewLine;
        }

        public string IndentText { get; set; }

        public string BlockStart { get; set; }
        public string BlockEnd { get; set; }

        public string BlockCommentStart { get; set; }
        public string BlockCommentEnd { get; set; }

        public string CommentStart { get; set; }
        public string CommentEnd { get; set; }

        public override Encoding Encoding
        {
            get { return _w.Encoding; }
        }

        public override string NewLine
        {
            get { return _w.NewLine; }
            set { SetNewLine(value); }
        }

        public override void Write(char value)
        {
            WriteIndent();
            if (value == _newline)
                _writeIndent = true;

            // normalize line endings...
            if (value == _newline)
                _w.Write(NewLine);
            else if (!_ignored.Contains(value))
                _w.Write(value);
        }

        /// <summary> Indents the code a level </summary>
        public IDisposable Indent()
        {
            _indentLevel++;
            return Disposable.Create(Dedent);
        }

        /// <summary> Writes a block </summary>
        /// <param name="trailer">text to immediatly follow the closing brace, before the newline. usually a ;</param>
        public IDisposable Block(string trailer = null)
        {
            WriteLine(BlockStart);
            var d = Indent();
            return Disposable.Create(() =>
            {
                d.Dispose();
                WriteLine($"{BlockEnd}{trailer}");
            });
        }

        /// <summary> Writes a block comment</summary>
        public IDisposable BlockComment()
        {
            WriteLine(BlockCommentStart);
            var d = Indent();
            return Disposable.Create(() =>
            {
                d.Dispose();
                WriteLine($"{BlockCommentEnd}");
            });
        }

        public void WriteComment(string comment)
        {
            Write(CommentStart);
            Write(comment.Replace(CommentEnd, $"{NewLine}{CommentStart}"));
            Write(CommentEnd);
        }

        public override void Flush()
        {
            _w.Flush();
        }

        public override string ToString()
        {
            return _w.ToString();
        }

        protected override void Dispose(bool disposing)
        {
            if (disposing)
            {
                Flush();
                _w.Dispose();
            }

            base.Dispose(disposing);
        }

        private void SetNewLine(string value)
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentOutOfRangeException(nameof(value), "Newline cannot be null or empty");
            _w.NewLine = value;
            _newline = value.ToCharArray().Last();
            _ignored = value.ToCharArray().Except(new[] {_newline}).ToArray();
        }

        private void Dedent()
        {
            _indentLevel--;
        }

        private void WriteIndent()
        {
            if (_writeIndent)
            {
                for (int i = 0; i < _indentLevel; i++)
                    _w.Write(IndentText);

                _writeIndent = false;
            }
        }
    }


```

