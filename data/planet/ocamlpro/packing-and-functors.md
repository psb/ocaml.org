---
title: Packing and Functors
description: We have recently worked on modifying the OCaml system to be able to pack
  a set of modules within a functor, parameterized on some signatures. This page presents
  this work, funded by Jane Street. All the patches on this page are provided for
  OCaml version 3.12.1. Packing Functors Installation of the ...
url: https://ocamlpro.com/blog/2011_08_10_packing_and_functors
date: 2011-08-10T13:19:46-00:00
preview_image: URL_de_votre_image
featured:
authors:
- "\n    Fabrice Le Fessant\n  "
source:
---

<p>We have recently worked on modifying the OCaml system to be able to
pack a set of modules within a functor, parameterized on some
signatures. This page presents this work, funded by Jane Street.</p>
<p>All the patches on this page are provided for OCaml version 3.12.1.</p>
<h2>Packing Functors</h2>
<h3>Installation of the modified OCaml system</h3>
<p>The patch for OCaml 3.12.1 is available here:</p>
<pre><code class="language-shell-session">ocaml+libfunctor-3.12.1.patch.gz (26 kB)
</code></pre>
<p>To use it, you can use the following recipe, that will compile and
install the patched version in <code>~/ocaml+libfunctor-3.12.1/bin/</code>.</p>
<pre><code class="language-shell-session">~% wget http://caml.inria.fr/pub/distrib/ocaml-3.12/ocaml-3.12.1.tar.gz
~% tar zxf ~/ocaml-3.12.1.tar.gz
~% cd ocaml-3.12.1
~/ocaml-3.12.1% wget ocamlpro.com/code/ocaml+libfunctor-3.12.1.patch.gz
~/ocaml-3.12.1% gzip -d ocaml+libfunctor-3.12.1.patch.gz
~/ocaml-3.12.1% patch -p1 &lt; ocaml+libfunctor-3.12.1.patch
~/ocaml-3.12.1% ./configure &ndash;prefix ~/ocaml+libfunctor-3.12.1
~/ocaml-3.12.1% make coldstart
~/ocaml-3.12.1% make ocamlc ocamllex ocamltools
~/ocaml-3.12.1% make library-cross
~/ocaml-3.12.1% make bootstrap
~/ocaml-3.12.1% make all opt opt.opt
~/ocaml-3.12.1% make install
~/ocaml-3.12.1% cd ~
~% export PATH=$HOME/ocaml+libfunctor-3.12.1/bin:$PATH
</code></pre>
<p>Note that it needs to bootstrap the compiler, as the format of object
files is not compatible with the one of ocaml-3.12.1.</p>
<h3>Usage of the lib-functor patch.</h3>
<p>Now that you are equiped with the new system, you can start using it. The lib-functor patch adds two new options to the compilers ocamlc and ocamlopt:</p>
<ul>
<li>
<p><code>-functor &lt;interface_file&gt;</code> : this option is used to specify that the current module is compiled with the interface files specifying the argument of the functor. This option should be used together with -for-pack <module>, where <module> is the name of the module in which the current module will be embedded.</module></module></p>
</li>
<li>
<p><code>-pack-functor &lt;module&gt;</code> : this option is used to pack the modules. It should be used with the option -o &lt;object_file&gt; to specify in which module it should be embedded. The <module> specified with -pack-functor specifies the name of functor that will be created in the target object file.</module></p>
</li>
</ul>
<p>If the interface x.mli contains :</p>
<pre><code class="language-ocaml">type t
val compare : t -&gt; t -&gt; int
</code></pre>
<p>and the files <code>xset.ml</code> and <code>xmap.ml</code> contain respectively :</p>
<pre><code class="language-ocaml">module T = Set.Make(X)
</code></pre>
<pre><code class="language-ocaml">module T = Map.Make(X)
</code></pre>
<p>Then :</p>
<pre><code class="language-shell-session">~/test% ocamlopt -c -for-pack Xx -functor x.cmi xset.ml
~/test% ocamlopt -c -for-pack Xx -functor x.cmi xmap.ml
~/test% ocamlopt -pack-functor MakeSetAndMap -o xx.cmx xset.cmx xmap.cmx
</code></pre>
<p>will construct a compiled unit whose signature is (that you can get
with <code>ocamlopt -i xx.cmi</code>, see below) :</p>
<pre><code class="language-ocaml">module MakeSetAndMap :
functor (X : sig type t val compare : t -&gt; t -&gt; int end) -&gt; sig
  module Xset : sig
    module T : sig
      type elt = X.t
      type t = Set.Make(X).t
      val empty : t
      val is_empty : t -&gt; bool
      &hellip;
    end
  end
  module Xmap : sig
    module T : sig
      type key = X.t
      type &lsquo;a t = &lsquo;a Map.Make(X).t
      val empty : &lsquo;a t
      val is_empty : &lsquo;a t -&gt; bool
      &hellip;
    end
  end
end
</code></pre>
<h3>Other extension: printing interfaces</h3>
<p>OCaml only allows you to print the interface of a module or interface
by compiling its source with the -i option. However, you don&rsquo;t always
have the source of an object interface (in particular, if it was
generated by packing), and you might still want to do it.</p>
<p>In such a case, the lib-functor patch allows you to do that, by using
the -i option on an interface object file:</p>
<pre><code class="language-shell-session">~/test% cat &gt; a.mli
val x : int
~/test% ocamlc -c -i a.mli
val x : int
~/test% ocamlc -c -i a.cmi
val x : int
</code></pre>
<h3>Other extension: packing interfaces</h3>
<p>OCaml only allows you to pack object files inside another object file
(.cmo or .cmx). When doing so, you can either provide an source
interface (.mli) that you need to compile to provide the corresponding
object interface (.cmi), or the object interface will be automatically
generated by exporting all the sub-modules within the packed module.</p>
<p>However, sometimes, you would want to be able to specify the
interfaces of each module separately, so that:</p>
<ul>
<li>
<p>you can reuse most of the interfaces you already specified</p>
</li>
<li>
<p>you can use a different interface for a module, that the one used to
compile the other modules. This happens when you want to export more
values to the other internal sub-modules than you want to export to
the user.</p>
</li>
</ul>
<p>In such a case, the lib-functor patch allows you to do that, by using
the -pack option on interface object files:</p>
<pre><code class="language-shell-session">test% cat &gt; a.mli
val x : int
test% cat &gt; b.mli
val y : string
test% ocamlc -c a.mli b.mli
test% ocamlc -pack -o c.cmi a.cmi b.cmi
test% ocamlc -i c.cmi
module A : sig val x : int end
module B : sig val y : string end
</code></pre>
<h2>Using <code>ocp-pack</code> to pack source files</h2>
<h3>Installation of ocp-pack</h3>
<p>Download the source file from:</p>
<p><code>ocp-pack-1.0.1.tar.gz</code> (20 kB, GPL Licence, Copyright OCamlPro SAS)</p>
<p>Then, you just need to compile it with:</p>
<pre><code class="language-shell-session">~% tar zxf ocp-pack-1.0.1.tar.gz
~% cd ocp-pack-1.0.1
~/ocp-pack-1.0.1% make
~/ocp-pack-1.0.1% make install
</code></pre>
<h3>Usage of <code>ocp-pack</code></h3>
<p><code>ocp-pack</code> can be used to pack source files of modules within just one
source file. It allows you to avoid the use of the <code>-pack</code> option, that
is not always supported by all ocaml tools (for example,
<code>ocamldoc</code>). Moreover, <code>ocp-pack</code> tries to provide the correct locations
to the compiler, so errors are not reported within the generated
source file, but within the original source files.</p>
<p>It supports the following options:</p>
<pre><code class="language-shell-session">% ocp-pack -help
Usage:
ocp-pack -o target.ml [options] files.ml*

Options:
-o &lt;filename.ml&gt; generate filename filename.ml
-rec use recursive modules
all .ml files must have a corresponding .mli file
-pack-functor &lt;modname&gt; create functor with name &lt;modname&gt;
-functor &lt;filename.mli&gt; use filename as an argument for functor
-mli output the .mli file too
.ml files without .mli file will not export any value
-no-ml do not output the .ml file
-with-ns use directory structure to create a hierarchy of modules
-v increment verbosity
&ndash;version display version information
</code></pre>
<p><code>ocp-pack</code> automatically detects interface sources and implementation
sources. When only the interface source is available, it is assumed
that it is a type-only module, i.e. no val items are present inside.</p>
<p>Here is an example of using <code>ocp-pack</code> to build the ocamlgraph package:</p>
<pre><code class="language-shell-session">test% ocp-pack -o graph.ml 
lib/bitv.ml lib/heap.ml lib/unionfind.ml 
src/sig.mli src/dot_ast.mli src/sig_pack.mli 
src/version.ml src/util.ml src/blocks.ml 
src/persistent.ml src/imperative.ml src/delaunay.ml 
src/builder.ml src/classic.ml src/rand.ml src/oper.ml 
src/path.ml src/traverse.ml src/coloring.ml src/topological.ml 
src/components.ml src/kruskal.ml src/flow.ml src/graphviz.ml 
src/gml.ml src/dot_parser.ml src/dot_lexer.ml src/dot.ml 
src/pack.ml src/gmap.ml src/minsep.ml src/cliquetree.ml 
src/mcs_m.ml src/md.ml src/strat.ml
test% ocamlc -c graph.ml
test% ocamlopt -c graph.ml
</code></pre>
<p>The -with-ns option can be used to automatically build a hierarchy of
modules. With that option, sub-directories are seen as
sub-modules. For example, packing a/x.ml, a/y.ml and b/z.ml will give
a result like:</p>
<p>[code language=&rdquo;fsharp&rdquo;]
module A = struct
module X = struct &hellip; end
module Y = struct &hellip; end
end
module B = struct
module Z = struct &hellip; end
end
[/code]</p>
<h3>Packing modules as functors</h3>
<p>The <code>-pack-functor</code> and <code>-functor</code> options provide the same behavior
as the same options with the lib-functor patch. The only difference is
that <code>-functor</code> takes the interface source as argument, not the
interface object.</p>
<h3>Packing recursive modules</h3>
<p>When trying to pack modules with <code>ocp-pack</code>, you might discover that
your toplevel modules have recursive dependencies. This is usually
achieved by types declared abstract in the interfaces, but depending
on each other in the implementations. Such modules cannot simply
packed by <code>ocp-pack</code>.</p>
<p>To handle them, <code>ocp-pack</code> provides a <code>-rec</code> option. With that option,
modules are put within a module rec construct, and are all required to
be accompagnied by an interface source file.</p>
<p>Moreover, in many cases, OCaml is not able to compile such recursive modules:</p>
<ul>
<li>
<p>For typing reasons: recursive modules are typed in an environment
containing only an approximation of other recursive modules
signatures</p>
</li>
<li>
<p>For code generation reasons: recursive modules can be reordered
depending on their shape, and this reordering can generate an order
that is actually not safe, leading to an exception at runtime</p>
</li>
</ul>
<p>To solve these two issues in most cases, you can use the following
patch (you can apply it using the same recipe as for lib-functor, and
even apply both patches on the same sources):</p>
<ul>
<li><code>ocaml+rec-3.12.1.patch.gz</code>
</li>
</ul>
<p>With this patch, recursive modules are typed in an environment that is
enriched progressively with the final types of the modules as soon as
they become available. Also, during code generation, a topological
order is computed on the recursive modules, and the subset of modules
that can be initialized using in that topological order are immediatly
generated, leaving only the other modules to be reordered.</p>
