# Build your own Resource Leak analysis

This is a lab exercise designed to take the participant through the basics of using Infer's Abstract Interpretation (AI) framework for building compositional abstract interpreters. We provide the skeleton for a simple intraprocedural resource leak analysis. During this exercise, you will identify limitations in the existing approach and work on extending it to create a more powerful interprocedural analysis.

The files to work on are [ResourceLeaks.ml](./ResourceLeaks.ml) and [ResourceLeakDomain.ml](./ResourceLeakDomain.ml), and their corresponding .mli files.

The solutions to the exercises can also be found in this directory, each in their own directory. For example, the solution for Section 2 of this lab can be found in [02_domain_join/](./02_domain_join/).

## (0) Quick Start

Using **VS Code Dev Containers** is the fastest way: everything is set up in the Docker image for you to start hacking on Infer straight away.

1. Get VS Code Dev Containers: https://code.visualstudio.com/docs/remote/containers

2. Clone this repository from https://github.com/sws-lab/digit2025-infer

3. Open the cloned repository in a Dev Container:

   1. After opening VS Code in the cloned directory, select "Reopen in Container" in the notification.

   2. If you miss the notification, you can always select "Remote-Containers: Reopen in Container" from the command palette.

   This will give you a VS Code instance *inside the Docker image* to use for the duration of the lab.

4. Build & install Infer with `make install` in `/workspaces/infer` (the default working directory in the Dev Container).

## (1) Warm up: running, testing, and debugging Infer

(a) Open a new terminal and change to the test directory (`cd infer/tests/codetoanalyze/java/lab`) and run infer in its default configuration:

```
infer -- javac Leaks.java
```

Infer should report 7 resource leaks. These reports come from the separation logic based biabduction analysis.

(b) Run the analyzer on a single test file to produce the debug HTML:

```
infer -g --resource-leak-lab-only -- javac Leaks.java
```

This produces debug HTML at `infer-out/captured/*.html`.
To view it, open another terminal and run:
```
python3 -m http.server
```
VS Code automatically sets up port forwarding between the Dev Container and the host, so you can just open http://0.0.0.0:8000/ in a browser running on the host.

This helpful artifact shows the Infer warnings alongside the code they are complaining about. It also displays the CFG node(s) associated with each line of code. Clicking a CFG node shows the Infer IR associated with the node, and the pre/post state computed while analyzing the instruction. Come back to the debug HTML early and often when you can't understand what your analysis is doing–it will help!

(c) The `Logging.d_printf`/`Logging.d_printfln` functions print to the debug HTML. The logging is contextual: it will print to the log for the CFG node that's being analyzed at the time the logging statement executes. Try adding `Logging.d_printfln "Hi Infer";` inside of the case for `Call` in [ResourceLeaks.ml](./ResourceLeaks.ml), recompiling/re-running. You should see the text you printed inside the appropriate CFG node log in the debug HTML.

(d) The `Logging.debug_dev` function prints to the console. This can be useful for printing information that doesn't happen in the context of a particular CFG node (e.g., performing post-processing on a summary). Try adding `Logging.debug_dev "Hi Infer@\n" [@alert "-deprecated"];` to the `compute_post` function, recompile/re-run and see your text printed on the command line.

Note: Using `Logging.debug_dev` generates warnings telling you that this function is deprecated. This is used to prevent developers from landing leftover debug instructions.

## (2) Integer domain for straight-line programs

(a) Let's start with a simple-minded analysis: count the number of open resources. Change `ResourceLeakDomain.t` to be of type `int`:

```OCaml
type t = int
```

You don't need to change `join` or `widen` yet, this will be done later. You also don't need to change ResourceLeakDomain.mli, only ResourceLeakDomain.ml.

Building Infer with just this change will fail with type errors.
Change the pretty-printing function to output the integer:

```ocaml
let pp fmt resources = F.fprintf fmt "%d" resources
```

You also need to change the value of `initial`.

(b) Now, in `ResourceLeaks.ml`, change the first `Call` case of `exec_instr` to bump the integer when a resource is acquired, and decrease it when a resource is released. Use `acquires_resource` (remove the leading `_`, telling OCaml not to warn that it was unused) and `releases_resource` (same). For the rest of the lab, it will be useful **not** to expose the type of `ResourceLeakDomain.t` to `ResourceLeaks.ml`, so add functions `ResourceLeakDomain.acquire_resource` and `ResourceLeakDomain.release_resource` to do the actual integer manipulations. Expose these functions in ResourceLeakDomain.mli`.

Finally, look again at the HTML debug output of infer on [Leaks.java](../../tests/codetoanalyze/java/lab/Leaks.java). You should see the resource count be incremented and decremented appropriately.

(c) Now let's report leaks! Write and expose a function `ResourceLeakDomain.has_leak`, true when an abstract state shows a leak. Then change `ResourceLeaks.report_if_leak` to report when `ResourceLeakDomain.has_leak post` is true.

(d) Think about the concretization of the resource count. What does a resource count of zero mean? Is there a concrete state in the concretization of "Resource count zero" that leaks a resource? Write a simple test method `FN_leakBad` in [Leaks.java](../../tests/codetoanalyze/java/lab/Leaks.java) that will produce this concrete state (that is, a false negative test where the program leaks a resource, but the analyzer doesn't catch it).

(e) In addition, there are programs that do not leak resources that the analyzer will flag. Write a simple test method `FP_noLeakOk` that exhibits this problem (that is, a false positive test that demonstrates the imprecision of the analyzer).

(f) Do your false negative and false positive examples have a common underlying cause? Can the domain be improved to address them?

Let's stick with just an integer domain to keep things simple until (5).

## (3) Integer domain for branching and looping programs

(a) Run your checker on [LeaksBranch.java](../../tests/codetoanalyze/java/lab/LeaksBranch.java). Uh oh, it crashed! Fix it by implementing `join` for your domain.

(b) Now run on [LeaksLoop.java](../../tests/codetoanalyze/java/lab/LeaksLoop.java), and implement `widen`. Hmm... There's a termination bug in the abstract domain–do you see it?

(c) Time for a richer domain! A classic solution to this problem in abstract interpretation is to add a `Top` value that is larger than any integer.

- Fix the domain to make your test pass and ensure that the analyzer will always terminate. Make sure your fix is sufficiently general (remember: the allocation count can be negative :)).

- Hint: `AbstractDomain.TopLifted` may be useful for this. Just put all the code in ResourceLeakDomain.ml that corresponds to the signature `AbstractDomain.S` into a submodule `FiniteBounds`, then let `TopLifted` do all the lifting with

```OCaml
include AbstractDomain.TopLifted (FiniteBounds)
```

- Hint#2: use `open AbstractDomain.Types` to be able to write, e.g., `Top` instead of `AbstractDomain.Top`.


## (4) Interprocedural analysis

Augment the summary type with state to indicate whether the current procedure returns a resource. Allowing a resource to escape to the caller should not be considered a leak. Use this information in callers too by reading from the callee's summary. Use the `analyze_dependency` field of the `InterproceduralAnalysis.t` record passed to the analysis like so:

```OCaml
  match analyze_dependency callee_procname with
  | Some callee_summary ->
    (* interprocedural analysis produced a summary: use it *)
    ()
  | None ->
    (* No summary for [callee_procname]; it's native code or missing for some reason *)
    ()
```

This improvement should allow your analysis to pass the tests in [LeaksInterprocedural.java](../../tests/codetoanalyze/java/lab/LeaksInterprocedural.java).

Hint: What do return values look like in infer? They are assignments to a special variable, detected by `PVar.is_return`. You may also want to match only return variables of some specific object type. Use this code to look at the classname of the type of the return value:

```OCaml
    | Store {e1= Lvar ret; e2= _rhs; typ={ Typ.desc= Typ.Tptr ({Typ.desc= Typ.Tstruct ret_typename}, _)}; loc= _loc}
      when PVar.is_return ret ->
```

Then `ret_var` is the return variable (if `PVar.is_return ret_var` returns true), of type `ret_typename` (really `ret_typename*` in infer's intermediate representation).


## (5) Access paths

(a) Change the simple counting domain to a domain that overapproximates the set of storage locations that hold a resource. As a concrete goal, the new domain should allow you to print the name of the resource(s) that leak in the error message (rather than just the number of resources). The new domain should also allow your analysis to get the correct answer on your false negative and false positive tests from 2(d) and 2(e). Think about the following questions when designing your new domain:

- How should we abstract storage locations? Is abstracting the stack program variables (`Var.t`) enough, or do we need an abstraction of the heap as well?

- How will we handle aliasing (storage locations that store the same address)? More precisely, how to handle assignement? We will need to make some simplifying assumptions and cut some corners here. Developing a full memory model that accounts for aliasing perfectly is out of the scope of this lab (check out how the Pulse analysis does it!).

- Will it be easy to extend the domain to incorporate information from the callee summaries/represent state that will be instantiated by callers?

- Some modules that might be useful in creating your new domain (depending on what approach you choose): `AbstractDomain.FiniteSet`, `AbstractDomain.Map`, `AccessPath`, `Var`, `FormalMap`.

- It's okay for the domain to diverge in certain cases for the purpose of this lab. Can you write a method that would make your checker diverge?

(b) Write some tests that demonstrate the limitations of your new domain: both false positives (names prefixed with `FP_` and false negatives (prefixed with `FN_`). Add them to [LeaksAccessPaths.java](../../tests/codetoanalyze/java/lab/LeaksAccessPath.java).

(c) Augment the summary type with state to indicate formals that are closed by the current function. This should allow your analysis to pass the tests in [LeaksAccessPathsInterprocedural.java](../../tests/codetoanalyze/java/lab/LeaksAccessPathsInterprocedural.java).

Hint: You will find the `FormalMap` module useful for this. This module lets you go back and forth between the index of a formal and its name. This utility module is also used in the `RacerD` and `TaintAnalysis` modules.


## (6) Making it practical

(a) Real resource leaks frequently involve failing to close resources along exceptional control-flow paths. For simplicity, the initial version of the current analysis uses a filtered view of the CFG that skips exceptional edges (`ProcCfg.Normal`). To find more bugs, you might want to switch to using `ProcCfg.Exceptional` and make sure that your analysis gets the right answer on some realistic exception examples like [LeaksExceptions.java](../../tests/codetoanalyze/java/lab/LeaksExceptions.java).

(b) Try running on real code! The instructions [here](http://fm.csl.sri.com/SSFT17/infer-instr.html) have several suggestions for open-source Android apps to point your analysis at. Try `./gradlew assembleDebug -x test` first to make sure everything builds correctly without Infer (if not, you are probably missing some dependencies--the error messages should guide you). Once that's working, try
`./gradlew clean; infer run --resource-leak-lab-only -- ./gradlew assembleDebug -x test`.
- Found a real bug? Bonus points! Send a pull request to fix it! Very frequently, the best fix is to use try-with-resources.
- Found a false positive in your analysis? Try re-running Infer with `--debug` and see if you can narrow down the root cause/fix it.
- How does your analysis compare to Infer's production resource leak analysis? Run with `infer -- <gradle command>` to see if your analysis finds bugs that Infer misses, or vice versa.
