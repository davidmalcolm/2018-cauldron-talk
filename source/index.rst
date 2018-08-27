.. Note on building:

   sphinx 1.6+ is incompatible with hieroglyph:
     https://github.com/nyergler/hieroglyph/issues/124
     https://github.com/nyergler/hieroglyph/issues/127

   As a workaround, I've been building this using a virtualenv
   containing sphinx 1.5.6:

     (in /home/david/nomad-coding):
       virtualenv venv-sphinx-1.5
       source venv-sphinx-1.5/bin/activate
       easy_install sphinx==1.5.6
       easy_install hieroglyph

   Activating the virtualenv:

   $ source /home/david/nomad-coding/venv-sphinx-1.5/bin/activate

   "make slides" then works

====================
Optimization records
====================

Showing the user what GCC's optimization passes are doing

.. TODO: ^^^ make this a subheading of the title or whatnot

GNU Tools Cauldron 2018

David Malcolm <dmalcolm@redhat.com>

https://dmalcolm.fedorapeople.org/presentations/cauldron-2018/

.. Abstract:

   How does an advanced end-user figure out what GCC's optimization passes
   are doing to their code, and how they might tweak things for speed?

   How do we (as GNU toolchain developers) debug and improve the optimization
   passes?

   I'll be talking about:

   * the existing approaches here and their limitations
   * experiments I've been doing to address these limitations by capturing
     "optimization records" in a machine-readable format
   * what this might enable, and
   * what this might mean for GCC middle-end maintainers.

.. When and where:
     Saturday, September 8, 15:30-16:30 Great Hall

.. TODO: objectives for the talk?



.. Better capturing of the existing dumps

.. Better dump messages

   opt_problem

     if nothing else, better locations for the "failure" message

   rich vectorization hints

What this talk is about
=======================

.. FIXME


Getting information from the optimizer
======================================

How do advanced users ask for more information on what GCC's optimizers
are doing?

e.g.

* "Why isn't this loop being vectorized?"

* "Did this function get inlined?  Why? / Why not?"

etc

.. TODO: but what about profiling?
   Need to figure out what's slow first
   Disconnect between profiling and optimization dumps


How does the user get *actionable* information?
===============================================

* "How do I make my code faster?"

  * "What stopped the loop getting vectorized?"

* "Is there some option I can turn on that's likely to help?"

* How can we (GCC devs) improve a particular optimization pass?

.. ...................................................................
.. The status quo
.. ...................................................................

The two existing approaches
===========================

GCC <= 8:

  * ``-fdump-tree-all -fdump-ipa-all -fdump-rtl-all``

    * examine ``foo.c.SOMETHING``

      * where "SOMETHING" is undocumented, and changes from revision to
        revision of the compiler (e.g. "foo.c.029t.einline")

    * no easy way to parse (both for humans and scripts)

    * what is important, and what isn't?

      * e.g. "only tell me about the hot loops"

.. nextslide::
   :increment:

* ``-fopt-info``

  * e.g. ``-fopt-info-all``, ``-fopt-info-vec-missed``

Example
=======

.. code-block:: c++

  #include <vector>

  std::size_t
  f(std::vector<std::vector<float>> const & v)
  {
    std::size_t ret = 0;
    for (auto const & w: v)
      ret += w.size();
    return ret;
  }

(from Freenode's #gcc; thanks Nightstrike)


Does it vectorize?
==================

gcc 8, with ``-fopt-info-all``, at ``-O3``

.. code-block:: none

  Analyzing loop at demo.cc:7
  demo.cc:7:24: note: ===== analyze_loop_nest =====
  demo.cc:7:24: note: === vect_analyze_loop_form ===
  demo.cc:7:24: note: === get_loop_niters ===
  demo.cc:7:24: note: Symbolic number of iterations is (((((unsigned long) _10 - (unsigned long) _9) - 24) /[ex] 8) * 768614336404564651 & 2305843009213693951) + 1
  demo.cc:7:24: note: === vect_analyze_data_refs ===
  demo.cc:7:24: note: got vectype for stmt: _7 = MEM[(float * *)SR.8_23];
  vector(2) long unsigned int
  demo.cc:7:24: note: got vectype for stmt: _8 = MEM[(float * *)SR.8_23 + 8B];
  vector(2) long unsigned int
  demo.cc:7:24: note: === vect_analyze_scalar_cycles ===
  demo.cc:7:24: note: Analyze phi: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: Access function of PHI: {0, +, _16}_1
  demo.cc:7:24: note: step: _16,  init: 0
  demo.cc:7:24: note: step unknown.
  demo.cc:7:24: note: Analyze phi: SR.8_23 = PHI <_9(5), _12(6)>
  demo.cc:7:24: note: Access function of PHI: {_9, +, 24}_1
  demo.cc:7:24: note: step: 24,  init: _9
  demo.cc:7:24: note: Detected induction.

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: Analyze phi: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: detected reduction: ret_6 = _16 + ret_17;
  demo.cc:7:24: note: Detected reduction.
  demo.cc:7:24: note: === vect_pattern_recog ===
  demo.cc:7:24: note: vect_is_simple_use: operand _16
  demo.cc:7:24: note: def_stmt: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: vect_is_simple_use: operand _16
  demo.cc:7:24: note: def_stmt: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: vect_is_simple_use: operand _15
  demo.cc:7:24: note: def_stmt: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: vect_is_simple_use: operand _16
  demo.cc:7:24: note: def_stmt: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: === vect_analyze_data_ref_accesses ===
  demo.cc:7:24: note: Detected interleaving load MEM[(float * *)SR.8_23] and MEM[(float * *)SR.8_23 + 8B]
  demo.cc:7:24: note: Detected interleaving load of size 3 starting with _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: There is a gap of 1 elements after the group

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: === vect_mark_stmts_to_be_vectorized ===
  demo.cc:7:24: note: init: phi relevant? ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: init: phi relevant? SR.8_23 = PHI <_9(5), _12(6)>
  demo.cc:7:24: note: init: stmt relevant? _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: init: stmt relevant? _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: init: stmt relevant? _14 = _8 - _7;
  demo.cc:7:24: note: init: stmt relevant? _15 = _14 /[ex] 4;
  demo.cc:7:24: note: init: stmt relevant? _16 = (long unsigned int) _15;
  demo.cc:7:24: note: init: stmt relevant? ret_6 = _16 + ret_17;
  demo.cc:7:24: note: vec_stmt_relevant_p: used out of loop.
  demo.cc:7:24: note: vect_is_simple_use: operand _16
  demo.cc:7:24: note: def_stmt: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: vec_stmt_relevant_p: stmt live but not relevant.
  demo.cc:7:24: note: mark relevant 1, live 1: ret_6 = _16 + ret_17;
  demo.cc:7:24: note: init: stmt relevant? _12 = SR.8_23 + 24;
  demo.cc:7:24: note: init: stmt relevant? if (_10 != _12)
  demo.cc:7:24: note: worklist: examine stmt: ret_6 = _16 + ret_17;
  demo.cc:7:24: note: vect_is_simple_use: operand _16
  demo.cc:7:24: note: def_stmt: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: type of def: internal

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: mark relevant 1, live 0: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: vect_is_simple_use: operand ret_17
  demo.cc:7:24: note: def_stmt: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: type of def: reduction
  demo.cc:7:24: note: mark relevant 1, live 0: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: worklist: examine stmt: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: vect_is_simple_use: operand 0
  demo.cc:7:24: note: vect_is_simple_use: operand ret_6
  demo.cc:7:24: note: def_stmt: ret_6 = _16 + ret_17;
  demo.cc:7:24: note: type of def: reduction
  demo.cc:7:24: note: reduc-stmt defining reduc-phi in the same nest.
  demo.cc:7:24: note: worklist: examine stmt: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: vect_is_simple_use: operand _15
  demo.cc:7:24: note: def_stmt: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: mark relevant 1, live 0: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: worklist: examine stmt: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: vect_is_simple_use: operand _14
  demo.cc:7:24: note: def_stmt: _14 = _8 - _7;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: mark relevant 1, live 0: _14 = _8 - _7;

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: worklist: examine stmt: _14 = _8 - _7;
  demo.cc:7:24: note: vect_is_simple_use: operand _8
  demo.cc:7:24: note: def_stmt: _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: mark relevant 1, live 0: _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: vect_is_simple_use: operand _7
  demo.cc:7:24: note: def_stmt: _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: mark relevant 1, live 0: _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: worklist: examine stmt: _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: worklist: examine stmt: _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: === vect_analyze_data_ref_dependences ===
  demo.cc:7:24: note: === vect_determine_vectorization_factor ===
  demo.cc:7:24: note: ==> examining phi: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: get vectype for scalar type:  size_t
  demo.cc:7:24: note: vectype: vector(2) long unsigned int
  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining phi: SR.8_23 = PHI <_9(5), _12(6)>
  demo.cc:7:24: note: ==> examining statement: _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: get vectype for scalar type:  float *
  demo.cc:7:24: note: vectype: vector(2) long unsigned int

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining statement: _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: get vectype for scalar type:  float *
  demo.cc:7:24: note: vectype: vector(2) long unsigned int
  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining statement: _14 = _8 - _7;
  demo.cc:7:24: note: get vectype for scalar type:  long int
  demo.cc:7:24: note: vectype: vector(2) long int
  demo.cc:7:24: note: get vectype for scalar type:  long int
  demo.cc:7:24: note: vectype: vector(2) long int
  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining statement: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: get vectype for scalar type:  long int
  demo.cc:7:24: note: vectype: vector(2) long int
  demo.cc:7:24: note: get vectype for scalar type:  long int
  demo.cc:7:24: note: vectype: vector(2) long int
  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining statement: _16 = (long unsigned int) _15;
  demo.cc:7:24: note: get vectype for scalar type:  long unsigned int
  demo.cc:7:24: note: vectype: vector(2) long unsigned int
  demo.cc:7:24: note: get vectype for scalar type:  long unsigned int

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: vectype: vector(2) long unsigned int
  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining statement: ret_6 = _16 + ret_17;
  demo.cc:7:24: note: get vectype for scalar type:  size_t
  demo.cc:7:24: note: vectype: vector(2) long unsigned int
  demo.cc:7:24: note: get vectype for scalar type:  size_t
  demo.cc:7:24: note: vectype: vector(2) long unsigned int
  demo.cc:7:24: note: nunits = 2
  demo.cc:7:24: note: ==> examining statement: _12 = SR.8_23 + 24;
  demo.cc:7:24: note: skip.
  demo.cc:7:24: note: ==> examining statement: if (_10 != _12)
  demo.cc:7:24: note: skip.
  demo.cc:7:24: note: vectorization factor = 2
  demo.cc:7:24: note: === vect_analyze_slp ===
  demo.cc:7:24: note: === vect_make_slp_decision ===
  demo.cc:7:24: note: === vect_analyze_data_refs_alignment ===
  demo.cc:7:24: note: recording new base alignment for _9
  demo.cc:7:24: note:   alignment:    8
  demo.cc:7:24: note:   misalignment: 0
  demo.cc:7:24: note:   based on:     _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: vect_compute_data_ref_alignment:

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: can't force alignment of ref: MEM[(float * *)SR.8_23]
  demo.cc:7:24: note: vect_compute_data_ref_alignment:
  demo.cc:7:24: note: can't force alignment of ref: MEM[(float * *)SR.8_23 + 8B]
  demo.cc:7:24: note: === vect_prune_runtime_alias_test_list ===
  demo.cc:7:24: note: === vect_enhance_data_refs_alignment ===
  demo.cc:7:24: note: vector alignment may not be reachable
  demo.cc:7:24: note: vect_can_advance_ivs_p:
  demo.cc:7:24: note: Analyze phi: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: reduc or virtual phi. skip.
  demo.cc:7:24: note: Analyze phi: SR.8_23 = PHI <_9(5), _12(6)>
  demo.cc:7:24: note: Vectorizing an unaligned access.
  demo.cc:7:24: note: === vect_analyze_loop_operations ===
  demo.cc:7:24: note: examining phi: ret_17 = PHI <0(5), ret_6(6)>
  demo.cc:7:24: note: examining phi: SR.8_23 = PHI <_9(5), _12(6)>
  demo.cc:7:24: note: ==> examining statement: _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: vect_is_simple_use: operand MEM[(float * *)SR.8_23]
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: vect_is_simple_use: operand MEM[(float * *)SR.8_23]
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: no array mode for V2DI[3]
  demo.cc:7:24: note: Data access with gaps requires scalar epilogue loop
  demo.cc:7:24: note: can't use a fully-masked loop because the target doesn't have the appropriate masked load or store.
  demo.cc:7:24: note: vect_model_load_cost: strided group_size = 3 .
  demo.cc:7:24: note: vect_model_load_cost: unaligned supported by hardware.
  demo.cc:7:24: note: vect_model_load_cost: inside_cost = 36, prologue_cost = 0 .
  demo.cc:7:24: note: ==> examining statement: _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: vect_is_simple_use: operand MEM[(float * *)SR.8_23 + 8B]
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: vect_is_simple_use: operand MEM[(float * *)SR.8_23 + 8B]
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: no array mode for V2DI[3]
  demo.cc:7:24: note: Data access with gaps requires scalar epilogue loop
  demo.cc:7:24: note: vect_model_load_cost: unaligned supported by hardware.
  demo.cc:7:24: note: vect_model_load_cost: inside_cost = 12, prologue_cost = 0 .
  demo.cc:7:24: note: ==> examining statement: _14 = _8 - _7;
  demo.cc:7:24: note: vect_is_simple_use: operand _8
  demo.cc:7:24: note: def_stmt: _8 = MEM[(float * *)SR.8_23 + 8B];
  demo.cc:7:24: note: type of def: internal

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: vect_is_simple_use: operand _7
  demo.cc:7:24: note: def_stmt: _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: === vectorizable_operation ===
  demo.cc:7:24: note: vect_model_simple_cost: inside_cost = 4, prologue_cost = 0 .
  demo.cc:7:24: note: ==> examining statement: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: vect_is_simple_use: operand _14
  demo.cc:7:24: note: def_stmt: _14 = _8 - _7;
  demo.cc:7:24: note: type of def: internal
  demo.cc:7:24: note: vect_is_simple_use: operand 4
  demo.cc:7:24: note: op not supported by target.
  demo.cc:7:24: note: not vectorized: relevant stmt not supported: _15 = _14 /[ex] 4;
  demo.cc:7:24: note: bad operation or unsupported loop bound.
  demo.cc:4:1: note: vectorized 0 loops in function.
  demo.cc:4:1: note: ===vect_slp_analyze_bb===
  demo.cc:7:24: note: === vect_analyze_data_refs ===
  demo.cc:7:24: note: got vectype for stmt: _9 = MEM[(struct vector * *)v_4(D)];
  vector(2) long unsigned int
  demo.cc:7:24: note: got vectype for stmt: _10 = MEM[(struct vector * *)v_4(D) + 8B];
  vector(2) long unsigned int
  demo.cc:7:24: note: === vect_analyze_data_ref_accesses ===

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: Detected interleaving load MEM[(struct vector * *)v_4(D)] and MEM[(struct vector * *)v_4(D) + 8B]
  demo.cc:7:24: note: Detected interleaving load of size 2 starting with _9 = MEM[(struct vector * *)v_4(D)];
  demo.cc:7:24: note: not vectorized: no grouped stores in basic block.
  demo.cc:7:24: note: ===vect_slp_analyze_bb===
  demo.cc:7:24: note: ===vect_slp_analyze_bb===
  demo.cc:7:24: note: === vect_analyze_data_refs ===
  demo.cc:7:24: note: got vectype for stmt: _7 = MEM[(float * *)SR.8_23];
  vector(2) long unsigned int
  demo.cc:7:24: note: got vectype for stmt: _8 = MEM[(float * *)SR.8_23 + 8B];
  vector(2) long unsigned int
  demo.cc:7:24: note: === vect_analyze_data_ref_accesses ===
  demo.cc:7:24: note: Detected interleaving load MEM[(float * *)SR.8_23] and MEM[(float * *)SR.8_23 + 8B]
  demo.cc:7:24: note: Detected interleaving load of size 2 starting with _7 = MEM[(float * *)SR.8_23];
  demo.cc:7:24: note: not vectorized: no grouped stores in basic block.
  demo.cc:7:24: note: ===vect_slp_analyze_bb===
  demo.cc:7:24: note: ===vect_slp_analyze_bb===
  demo.cc:7:24: note: ===vect_slp_analyze_bb===
  demo.cc:9:10: note: === vect_analyze_data_refs ===
  demo.cc:9:10: note: not vectorized: not enough data-refs in basic block.

.. nextslide::
   :increment:

The pertinent information was two slides ago.

It's easier to see with ``-fopt-info-optimized-missed``:

.. code-block:: none

  demo.cc:7:24: note: step unknown.
  demo.cc:7:24: note: vector alignment may not be reachable
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: no array mode for V2DI[3]
  demo.cc:7:24: note: Data access with gaps requires scalar epilogue lo
  op
  demo.cc:7:24: note: can't use a fully-masked loop because the target
  doesn't have the appropriate masked load or store.
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: no array mode for V2DI[3]
  demo.cc:7:24: note: Data access with gaps requires scalar epilogue lo
  op
  demo.cc:7:24: note: op not supported by target.
  demo.cc:7:24: note: not vectorized: relevant stmt not supported:  _15
  = _14 /[ex] 4;
  demo.cc:7:24: note: bad operation or unsupported loop bound.
  demo.cc:7:24: note: not vectorized: no grouped stores in basic block.
  demo.cc:7:24: note: not vectorized: no grouped stores in basic block.
  demo.cc:9:10: note: not vectorized: not enough data-refs in basic blo
  ck.

i.e.:

.. code-block:: none

  demo.cc:7:24: note: not vectorized: relevant stmt not supported:
  _15 = _14 /[ex] 4;

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: not vectorized: relevant stmt not supported:
  _15 = _14 /[ex] 4;

So we know that the failure is due to a (then) unsupported tree code.

But that doesn't tell us the location of the problematic statement:

It's using the location of the loop for (almost) everything:

.. code-block:: none

  demo.cc:7:24:
    for (auto const & w: v)
                          ^


Other problems
==============

This is just one loop.

There's no way to request information for just one loop, or to prioritize
the dumps by code "hotness".


Two kinds of improvement
========================

* Better output format for the messages we have

* Better messages


Our optimization dumping APIs
=============================

.. dump_*

.. and some use of fprintf

.. TODO: the problem with using "vect_location" for everything


Comparison with clang
=====================

TODO

clang::

  -fsave-optimization-record -foptimization-record-file=foo.yaml

(perhaps with a compatible format?  they have viewers)


We don't want a parallel API
============================

We don't want to repeat ourselves when dumping e.g.:

.. code-block:: c++

  if (dump_file && (dump_flags & TDF_DETAILS))
    {
      fprintf (dump_file,
               "can't frobnicate this stmt:\n");
      print_gimple_stmt (dump_file, stmt, 0, 0);
    }
  remark (loop_stmt, OPT_remark_foo,
          "can't frobnicate this stmt");


What I've done so far for GCC 9
===============================

``-fsave-optimization-record``
==============================

* New in GCC 9

* machine-readable output format

* writes a ``demo.cc.opt-record.json`` file

Example of JSON output
======================

First, some metadata:

.. code-block:: json

  [
      {
          "format": "1",
          "generator": {
              "version": "9.0.0 20180829 (experimental)",
              "name": "GNU C++14",
              "pkgversion": "(GCC) ",
              "target": "x86_64-pc-linux-gnu"
          }
      },

.. nextslide::
   :increment:

Then all of the passes (so they can be referred back to):

.. code-block:: json

      [
        {
            "num": -1,
            "type": "gimple",
            "name": "*warn_unused_result",
            "id": "0x469e830",
            "optgroups": []
        },
        "[...etc...]",
        {
            "num": -1,
            "type": "rtl",
            "name": "*clean_state",
            "id": "0x46bccc0",
            "optgroups": []
        }
    ]

.. nextslide::
   :increment:

Then the dump messages, a list of objects like this:

.. code-block:: json

              {
                  "kind": "note",
                  "count": {
                      "quality": "guessed_local",
                      "value": 9.5563e+08
                  },
                  "location": {
                      "line": 7,
                      "file": "demo.cc",
                      "column": 24
                  },
                  "pass": "0x46b8ae0",
                  "impl_location": {
                      "line": 4367,
                      "file": "../../src/gcc/tree-vect-data-refs.c",
                      "function": "vect_analyze_data_refs"
                  },

.. nextslide::
   :increment:

.. code-block:: json

                  "function": "_Z1fRKSt6vectorIS_IfSaIfEESaIS1_EE",
                  "inlining_chain": [
                      {
                          "fndecl": "std::size_t f(const std::vector<std::vector<float> >&)"
                      }
                  ]

.. nextslide::
   :increment:

The text of the message itself is "marked up" with metadata:

.. code-block:: json

                   "message": [
                      "got vectype for stmt: ",
                      {
                          "location": {
                              "line": 8,
                              "file": "demo.cc",
                              "column": 18
                          },
                          "stmt": "_8 = MEM[(float * *)SR.16_23 + 8B];\n"
                      },
                      {
                          "expr": "vector(2) long unsigned int"
                      },
                      "\n"
                  ],

.. nextslide::
   :increment:

so that e.g. an HTML presentation might be:

.. code-block:: html

  <div class="message">got vectype for stmt:
    <div class="stmt">
      <a href="demo.cc#line-8">_8 = MEM[(float * *)SR.16_23 + 8B];\n"</a>
    </div>
    <div class="expr">vector(2) long unsigned int</div>
  <div>

Similarly, an IDE could make use of this in other ways.


Example of HTML report from the JSON output
===========================================

Dump API changes
================

Previously:

.. code-block:: c++

  extern void dump_printf_loc (dump_flags_t, source_location,
                               const char *, ...)
    ATTRIBUTE_PRINTF_3;

GCC 9:

.. code-block:: c++

  extern void dump_printf_loc (dump_flags_t, const dump_location_t &,
                               const char *, ...)
    ATTRIBUTE_GCC_DUMP_PRINTF (3, 0);


dump_location_t
===============

The dump API now takes a ``dump_location_t``, rather than a
``source_location``.

``dump_location_t``

  * contains ``dump_user_location_t``:

    * source information plus profile count

  * and ``dump_impl_location_t``:

    * the emission location in gcc source (__FILE__, __LINE__ and
      function name).

.. nextslide::
   :increment:

``dump_location_t`` can be created from ``gimple *`` and from ``rtx_insn *``.

Hence, rather than:

.. code-block:: c++

          dump_printf_loc (MSG_NOTE, gimple_location (stmt),
                           "This statement cannot be analyzed for "
                           "gridification\n");

we write:

.. code-block:: c++

          dump_printf_loc (MSG_NOTE, stmt,
                           "This statement cannot be analyzed for "
                           "gridification\n");

.. nextslide::
   :increment:

Rather than just:

.. code-block:: none

  user-code.c:20:5: This statement cannot be analyzed for gridification

we also now have:

  * **profile count** of the statement in question (allowing for prioritization,
    and filtering of optimization dumps for unimportant code)

  * **emission location metadata**:
      file == "gcc/omp-grid.c", line == 407,
      function == "grid_inner_loop_gridifiable_p"


ATTRIBUTE_GCC_DUMP_PRINTF
=========================

* GCC <= 8: ``dump_printf`` and ``dump_printf_loc``
  were ``printf`` under the covers

* GCC 9: they use pretty_printer, and support format codes appropriate
  to the middle-end.

.. nextslide::
   :increment:

* **%E** (`gimple *`)

  Equivalent to: ``dump_gimple_expr (MSG_*, TDF_SLIM, stmt, 0)``

* **%G** (`gimple *`)

  Equivalent to: ``dump_gimple_stmt (MSG_*, TDF_SLIM, stmt, 0)``

* **%T** (`tree`)

  Equivalent to: ``dump_generic_expr (MSG_*, arg, TDF_SLIM)``

All of this is supported in the JSON output (with the "markup" seen
earlier).

.. nextslide::
   :increment:

Hence it becomes possible to convert e.g.:

.. code-block:: c++

  if (dump_enabled_p ())
    {
      dump_printf_loc (MSG_MISSED_OPTIMIZATION, vect_location,
                       "not vectorized: different sized vector "
                       "types in statement, ");
      dump_generic_expr (MSG_MISSED_OPTIMIZATION, TDF_SLIM, vectype);
      dump_printf (MSG_MISSED_OPTIMIZATION, " and ");
      dump_generic_expr (MSG_MISSED_OPTIMIZATION, TDF_SLIM, nunits_vectype);
      dump_printf (MSG_MISSED_OPTIMIZATION, "\n");
    }

.. nextslide::
   :increment:

into:

.. code-block:: c++

  if (dump_enabled_p ())
    dump_printf_loc (MSG_MISSED_OPTIMIZATION, vect_location,
                     "not vectorized: different sized vector "
                     "types in statement, %T and %T\n",
                     vectype, nunits_vectype);

Question: should I go and clean up all dumps in our source tree to use the
new format code?

.. nextslide::
   :increment:

.. code-block:: c++

  if (dump_enabled_p ())
    {
      dump_printf_loc (MSG_NOTE, vect_location,
                       "\touter base_address: ");
      dump_generic_expr (MSG_NOTE, TDF_SLIM,
                         STMT_VINFO_DR_BASE_ADDRESS (stmt_info));
      dump_printf (MSG_NOTE, "\n\touter offset from base address: ");
      dump_generic_expr (MSG_NOTE, TDF_SLIM,
                         STMT_VINFO_DR_OFFSET (stmt_info));
      dump_printf (MSG_NOTE,
                   "\n\touter constant offset from base address: ");
      dump_generic_expr (MSG_NOTE, TDF_SLIM,
                         STMT_VINFO_DR_INIT (stmt_info));
      dump_printf (MSG_NOTE, "\n\touter step: ");
      dump_generic_expr (MSG_NOTE, TDF_SLIM,
                         STMT_VINFO_DR_STEP (stmt_info));
      dump_printf (MSG_NOTE, "\n\touter base alignment: %d\n",
                   STMT_VINFO_DR_BASE_ALIGNMENT (stmt_info));
      dump_printf (MSG_NOTE, "\n\touter base misalignment: %d\n",
                   STMT_VINFO_DR_BASE_MISALIGNMENT (stmt_info));
      dump_printf (MSG_NOTE, "\n\touter offset alignment: %d\n",
                   STMT_VINFO_DR_OFFSET_ALIGNMENT (stmt_info));
      dump_printf (MSG_NOTE, "\n\touter step alignment: %d\n",
                   STMT_VINFO_DR_STEP_ALIGNMENT (stmt_info));
    }

.. nextslide::
   :increment:

.. code-block:: c++

  if (dump_enabled_p ())
    dump_printf_loc (MSG_NOTE, vect_location,
		     "\touter base_address: %T"
		     "\n\touter offset from base address: %T"
		     "\n\touter constant offset from base address: %T"
		     "\n\touter step: %T"
		     "\n\touter base alignment: %d\n"
		     "\n\touter base misalignment: %d\n",
		     "\n\touter offset alignment: %d\n"
		     "\n\touter step alignment: %d\n"
		     STMT_VINFO_DR_BASE_ADDRESS (stmt_info),
		     STMT_VINFO_DR_OFFSET (stmt_info),
		     STMT_VINFO_DR_INIT (stmt_info),
		     STMT_VINFO_DR_STEP (stmt_info),
		     STMT_VINFO_DR_BASE_ALIGNMENT (stmt_info),
		     STMT_VINFO_DR_BASE_MISALIGNMENT (stmt_info),
		     STMT_VINFO_DR_OFFSET_ALIGNMENT (stmt_info),
		     STMT_VINFO_DR_STEP_ALIGNMENT (stmt_info));

Further ideas for GCC 9
=======================

* opt_problem

* rich vectorization hints


opt_problem
===========


Rich vectorization hints
========================

Focusing on "actionable" reports to end-user.

* Rich optimization hints can contain a mixture of:

  * text

  * diagrams

  * highlighted source locations/ranges,

  * proposed patches

  * etc.

.. nextslide::
   :increment:
   
* can be printed to stderr, or saved as part of the
  JSON optimization

  * can be prioritized by code hotness

  * browsed in an IDE, etc

etc.  The diagrams are printed as ASCII art when printed to stderr, or
serialized in a form from which HTML/SVG can be generated (this last
part is a work-in-progress).

.. nextslide::
   :increment:
   
.. code-block:: c++

  void
  my_example (int n, int *a, int *b, int *c)
  {
    int i;
  
    for (i=0; i<n; i++) {
      a[i] = b[i] + c[i];
    }
  }

.. nextslide::
   :increment:
   
.. code-block:: none

  ==[Loop vectorized]=================================================

  I was able to vectorize this loop, using SIMD instructions to reduce
  the number of iterations by a factor of 4.

  ../../src/vect-test.c:6:3:
     for (i=0; i<n; i++) {
     ^~~
  
                                     |
                             +--------------+
               +-------------|run-time tests|-----------+
               |             +--------------+           |
  +-------------------------+                +--------------------+
  |vectorized loop          |                |scalar loop         |
  |  iteration count: n / 4 |                |  iteration count: n|
  +-------------------------+                |                    |
               |                             |                    |
  +-------------------------+                |                    |
  |epilogue                 |                |                    |
  |  iteration count: [0..3]|                |                    |
  +-------------------------+                +--------------------+
               |                                        |
               +---------------------+------------------+
                                     |
  ------------------------------------------------[gcc.vect.success]--

.. nextslide::
   :increment:
   
.. code-block:: none

                                     |
                             +--------------+
               +-------------|run-time tests|-----------+
               |             +--------------+           |
  +-------------------------+                +--------------------+
  |vectorized loop          |                |scalar loop         |
  |  iteration count: n / 4 |                |  iteration count: n|
  +-------------------------+                |                    |
               |                             |                    |
  +-------------------------+                |                    |
  |epilogue                 |                |                    |
  |  iteration count: [0..3]|                |                    |
  +-------------------------+                +--------------------+
               |                                        |
               +---------------------+------------------+
                                     |
  ------------------------------------------------[gcc.vect.success]--
  
.. nextslide::
   :increment:
   
.. code-block:: none

  ==[Run-time aliasing check]=========================================
  
  Problem:
  I couldn't prove that these data references don't alias, so I had to
  add a run-time test, falling back to a scalar loop for when they do.
  
  Details:
  (1) This read/write pair could alias:
  
  ../../src/vect-test.c:7:13:
       a[i] = b[i] + c[i];
       ~^~~   ~^~~
  
  (2) This read/write pair could alias:
  
  ../../src/vect-test.c:7:20:
       a[i] = b[i] + c[i];
       ~^~~          ~^~~
  
.. nextslide::
   :increment:
   
.. code-block:: none
      
  Suggestion:
  If you know that the buffers cannot overlap in memory, marking them
  with restrict will allow me to assume it when optimizing this loop,
  and eliminate the run-time test.
  
  --- ../../src/vect-test.c
  +++ ../../src/vect-test.c
  @@ -1,5 +1,5 @@
   void
  -my_example (int n, int *a, int *b, int *c)
  +my_example (int n, int * restrict a, int * restrict b, int * restrict c)
   {
     int i;
  
  ---------------------[gcc.vect.loop-requires-versioning-for-alias]--

.. nextslide::
   :increment:
   
.. code-block:: none
      
  ==[Epilogue required for peeling]===================================
  
  Problem:
  I couldn't prove that the number of iterations is a multiple of 4,
  so I had to add an "epilogue" to cover the final 0-3 iterations.
  
  Details:
  FIXME: add a source code highlight or other visualization here?
  
  Suggestion:
  FIXME: add a suggestion here?
  ---------------------[gcc.vect.loop-requires-epilogue-for-peeling]--

.. nextslide::
   :increment:
 
Known limitations:

* lots of hacks: not all of the above is using real data (I'm currently
  generating this in try_vectorize_loop_1 after the loop_vinfo has been
  populated, but before the call to vect_transform_loop; it might make
  more sense to build it after vect_transform_loop, optionally capturing
  pertinent extra data if rich hints are enabled).
  
* no i18n yet
  
* diagrams don't yet support non-ASCII text
  
* various FIXMEs identified in the code
  
* this only covers the "loop vectorization succeeded" case (and only a
  subset of that).  It would be good to be able to emit rich hints
  describing the problem to the user when loop vectorization fails on a
  hot loop.  The "opt_problem" work I've posted elsewhere might be useful
  for that.


Summary
=======

.. FIXME


Next steps
==========

.. FIXME

Questions and Discussion
========================

Thanks for listening!

(thanks to Red Hat for funding this work)

URL for these slides: https://dmalcolm.fedorapeople.org/presentations/cauldron-2018/


TODO: Material under construction
=================================

.. TODO:
   What have I committed so far:

   "[PATCH 00/10] RFC: Prototype of compiler-assisted performance analysis"
   https://www.phoronix.com/scan.php?page=news_item&px=GCC-Comp-Assist-Perf-Analysis
   “[PATCH 0/8] v2 of optimization records patch kit"
   r261325: "Convert dump and optgroup flags to enums"
   r261710: "Introduce DUMP_VECT_SCOPE macro"
   "[PATCH] v3 of optinfo, remarks and optimization records"
   "[PATCH] [RFC] Higher-level reporting of vectorization problems"
   r262149: "Introduce dump_location_t"
   r262220: “Hide alt_dump_file within dumpfile.c"
   r262246: "-fopt-info: add indentation via DUMP_VECT_SCOPE"
   r262295: "Reinstate dump_generic_expr_loc"
   r262317: "selftest: introduce class auto_fix_quotes"
   "[PATCH 0/2] v4: optinfo framework and remarks"
   r262346: “Remove "note: " prefix from some scan-tree-dump directives"
   "[PATCH 0/5] [RFC v2] Higher-level reporting of vectorization problems"
   r262891: “Add "optinfo" framework" (v5)
   r262905: “Add "-fsave-optimization-record""
   "[PATCH] -fsave-optimization-record: add contrib/optrecord.py"
   r262950: “Fix segfault in -fsave-optimization-record (PR tree-optimization/86636)"
   r262967: "optinfo-emit-json.cc: fix trivial memory leak"
   "[PATCH] RFC: Prototype of "rich vectorization hints" idea"
   r263031: “Fixes to testcase for PR tree-optimization/86636"
   "[PATCH 0/5] dump_printf support for middle-end types"
   r263046: "C++: clean up cp_printer"
   r263167: "Simplify dump_context by adding a dump_loc member function"
   r263178: "dumpfile.c: eliminate special-casing of dump_file/alt_dump_file"
   r263181: "c-family: clean up the data tables in c-format.c"
   r263244: "dumpfile.c/h: add "const" to dump location ctors"
   r263626: "Formatted printing for dump_* in the middle-end"
      
.. TODO:
   What's left

.. What does it look like on SPEC?


How do advanced users ask for more information on what GCC's optimizers
are doing?

Current user experience

Some kind of profiling of the workload
Find the hotspots in the code
Scour over the machine code and the dumps

Clang experience

I guess I'm assuming PGO

Existing dump API.

The formatted version of the API
