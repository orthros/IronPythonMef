IronPythonMef
=============

Create plugins for .NET applications with IronPython via MEF, the Managed Extensibility Framework

This is a fork from a [project by Bruno Lopes](https://github.com/brunomlopes/ILoveLucene/tree/master/src/Plugins/IronPython). __Thanks, Bruno!__ This was just [what I needed](http://stackoverflow.com/questions/13337319/using-mef-to-import-components-exported-by-ironpython-or-other-dlr-languages).

# Problem

You want to write IronPython scripts to extend or create plugins for a .NET application. And, you want to `Export` types from IronPython / DLR to the CLR and `Import` types from the CLR. You've come to the right place. Keep reading.

IronPythonMef is the __solution__.

# Single File Example

1. Create a new C# console app
2. You can download the code from here, or use NuGet to [get the package](https://www.nuget.org/packages/IronPythonMef).
2. Replace the contents of `Program.cs` with the code below.
3. Run it!

```c#
using System;
using System.Collections.Generic;
using System.ComponentModel.Composition;
using System.ComponentModel.Composition.Hosting;
using System.ComponentModel.Composition.Primitives;
using System.Reflection;
using IronPython.Hosting;
using IronPythonMef;

namespace IronPythonMefInAMinute
{
    public interface IMessenger
    {
        string GetMessage();
    }

    public interface IConfig
    {
        string Intro { get; }
    }

    /// <summary>
    /// Gets exported from IronPython into the CLR Demo instance.
    /// </summary>
    public static class PythonScript
    {
        public static readonly string Code =
@"
@export(IMessenger)
class PythonMessenger(IMessenger):
    def GetMessage(self):
        return self.config.Intro + ' from IronPython'

    @import_one(IConfig)
    def import_config(self, config):
        self.config = config
";
    }

    /// <summary>
    /// Also gets exported into the Demo instance.
    /// </summary>
    [Export(typeof(IMessenger))]
    public class ClrMessenger : IMessenger
    {
        [Import(typeof(IConfig))]
        public IConfig Config { get; set; }

        public string GetMessage()
        {
            return Config.Intro + " from C#!";
        }
    }

    /// <summary>
    /// This will get imported into both the IronPython class and ClrMessenger.
    /// </summary>
    [Export(typeof(IConfig))]
    public class Config : IConfig
    {
        public string Intro
        {
            get { return "Hello"; }
        }
    }

    public class Demo
    {
        [ImportMany(typeof(IMessenger))]
        public IEnumerable<IMessenger> Messengers { get; set; }

        public Demo()
        {
            // Create IronPython
            var engine = Python.CreateEngine();
            var script = engine.CreateScriptSourceFromString(PythonScript.Code);

            // Configure the engine with types
            var typesYouWantPythonToHaveAccessTo = new[] { typeof(IMessenger), typeof(IConfig) };
            var typeExtractor = new ExtractTypesFromScript(engine);
            var exports = typeExtractor.GetPartsFromScript(script,
                typesYouWantPythonToHaveAccessTo);

            // Compose with MEF
            var catalog = new AssemblyCatalog(Assembly.GetExecutingAssembly());
            var container = new CompositionContainer(catalog);
            var batch = new CompositionBatch(exports, new ComposablePart[] { });
            container.Compose(batch);
            container.SatisfyImportsOnce(this);
        }

        public static void Main(string[] args)
        {
            var demo = new Demo();

            foreach (var messenger in demo.Messengers)
            {
                Console.WriteLine(messenger.GetMessage());
            }

            Console.Read();
        }
    }
}
```
The output is simply:

    Hello from IronPython
    Hello from C#!

# Longer Example

Code is [here](https://github.com/JogoShugh/IronPythonMef/tree/master/src/IronPythonMef.Tests/Example).

## Given this interface in .NET:

```c#
namespace IronPythonMef.Tests.Example.Operations
{
    public interface IOperation
    {
        object Execute(params object[] args);
        string Name { get; }
        string Usage { get; }
    }
}
``` 
## And this IronPython script:

```python
@export(IOperation)
class Fibonacci(IOperation):
    def Execute(self, n):
        n = int(n)
        if n == 0:
            return 0
        elif n == 1:
            return 1
        else:
            return self.Execute(n-1) + self.Execute(n-2)
    
    @property
    def Name(self):
        return "fib"

    @property
    def Usage(self):
        return "fib n -- calculates the nth Fibonacci number"

@export(IOperation)
class Absolute(IOperation):
    def Execute(self, n):
        n = float(n)
        if (n < 0):
            return -n
        return n
    
    @property
    def Name(self):
        return "abs"

    @property
    def Usage(self):
        return "abs f -- calculates the absolute value of f"
```
## And also this C# file:

```c#
﻿using System;
using System.ComponentModel.Composition;

namespace IronPythonMef.Tests.Example.Operations
{
    [Export(typeof(IOperation))]
    public class Power : IOperation
    {
        public object Execute(params object[] args)
        {
            if (args.Length < 2)
            {
                throw new ArgumentException(Usage, "args");
            }

            var x = Convert.ToDouble(args[0]);
            var y = Convert.ToDouble(args[1]);

            return Math.Pow(x, y);
        }

        public string Name
        {
            get { return "pow"; }
        }

        public string Usage
        {
            get { return "pow n, y -- calculates n to the y power"; }
        }
    }

    [Export(typeof(IOperation))]
    public class Factorial : IOperation
    {
        public object Execute(params object[] args)
        {
            if (args.Length < 1)
            {
                throw new ArgumentException(Usage, "args");
            }

            var n = Convert.ToInt32(args[0]);

            int i;
            long x = 1;
            for (i = n; i > 1; i--) x = x * i;

            return x;
        }

        public string Name
        {
            get { return "fac"; }
        }

        public string Usage
        {
            get { return "fac n -- calculates n!"; }
        }
    }
}
```
## When I run this NUnit test:

```c#
using System.Linq;
using IronPythonMef.Tests.Example.Operations;
using NUnit.Framework;

namespace IronPythonMef.Tests.Example
{
    [TestFixture]
    public class MathWizardTests
    {
        [Test]
        public void runs_script_with_operations_from_both_csharp_and_python()
        {
            var mathWiz = new MathWizard();

            new CompositionHelper().ComposeWithTypesExportedFromPythonAndCSharp(
                mathWiz,
                "Operations.Python.py",
                typeof(IOperation));

            const string mathScript =
@"fib 6
fac 6
abs -99
pow 2 4
";
            var results = mathWiz.ExecuteScript(mathScript).ToList();

            Assert.AreEqual(8, results[0]);
            Assert.AreEqual(720, results[1]);
            Assert.AreEqual(99f, results[2]);
            Assert.AreEqual(16m, results[3]);
        }
    }
}
```
## Then, the test will __pass__!

# How to load an IronPython script and inject .NET interfaces into it so that your .NET app can import its exports

__Whoah, that sounds like _crazy talk_!__ Really? Not anymore! Here's how the unit test does it:

## CompositionHelper.cs:

```c#
using System;
using System.ComponentModel.Composition;
using System.ComponentModel.Composition.Hosting;
using System.ComponentModel.Composition.Primitives;
using System.IO;
using System.Linq;
using System.Reflection;
using IronPython.Hosting;
using Microsoft.Scripting.Hosting;

namespace IronPythonMef.Tests.Example
{
    public class CompositionHelper
    {
        public void ComposeWithTypesExportedFromPythonAndCSharp(
            object compositionTarget,
            string scriptsToImport,
            params Type[] typesToImport)
        {
            ScriptSource script;
            var engine = Python.CreateEngine();
            using (var scriptStream = GetType().Assembly.
                GetManifestResourceStream(GetType(), scriptsToImport))
            using (var scriptText = new StreamReader(scriptStream))
            {
                script = engine.CreateScriptSourceFromString(scriptText.ReadToEnd());
            }

            var typeExtractor = new ExtractTypesFromScript(engine);
            var exports = typeExtractor.GetPartsFromScript(script, typesToImport).ToList();

            var catalog = new AssemblyCatalog(Assembly.GetExecutingAssembly());

            var container = new CompositionContainer(catalog);
            var batch = new CompositionBatch(exports, new ComposablePart[] { });
            container.Compose(batch);
            
            container.SatisfyImportsOnce(compositionTarget);
        }
    }
}
```
In the code from the unit test, the `"Operations.Python.py"` is an embedded resource. It could also be a script on disk.

Note that Bruno Lopes has some more sophisticated examples in his code base, [such as import-on-start, and recomposition when files change or are added](https://github.com/brunomlopes/ILoveLucene/blob/master/src/Plugins/IronPython/IronPythonCommandsMefExport.cs). If I can find time or get the assistance to do so, I'll incorporate similar features into this.

```c#
new CompositionHelper().ComposeWithTypesExportedFromPythonAndCSharp(
    mathWiz,
    "Operations.Python.py",
    typeof(IOperation));
```
# Importing from the CLR into IronPython

This works too. It's not shown above, but the test cases and the MathWizard example has it.

Suppose you wanted to inject constants into IronPython, or other applications, to have a consistent approximation of `Pi`, or whatever.

## Definition and implementation in .NET

```c#
namespace IronPythonMef.Tests.Example.Operations
{
    public interface IMathCheatSheet
    {
        float Pi { get; set; }
    }
}

using System.ComponentModel.Composition;

namespace IronPythonMef.Tests.Example.Operations
{
    [Export(typeof(IMathCheatSheet))]
    public class MathCheatSheet : IMathCheatSheet
    {
        public MathCheatSheet()
        {
            Pi = 3.141592653589793f;
        }

        public float Pi { get; set; }
    }
}
```

## Import into IronPython via the `@import_one` decorator

```python
@export(IOperation)
class Circumference(IOperation):
    @import_one(IMathCheatSheet)
    def import_cheatSheet(self, cheatSheet):
        self.cheatSheet = cheatSheet

    def Execute(self, d):
        d = float(d)
        return self.cheatSheet.Pi * d

    @property
    def Name(self):
        return "crc"

    @property 
    def USage(self):
        return "crc d -- calculaets the circumference of a circle with diameter d"
```

So, this last example demonstrates that even though `Circumference` is itself exported from IronPython, it first gets its own import dependencies satisfied. Pretty awesome. All credit to the MEF team and Bruno on this.

# Resources

* MEF [home page](http://mef.codeplex.com/)
* Great [slides and examples](http://codebetter.com/glennblock/2010/06/13/way-of-mef-slides-and-code/) from Glenn Block
* IronPython [home page](http://ironpython.net/)
* Try IronPython inside your web browser, [right now](http://ironpython.net/try/)!