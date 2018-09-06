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

   See: http://docs.hieroglyph.io/en/latest/

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

====================
Optimization records
====================


Getting information from the optimizer
======================================

How do advanced users ask for more information on what GCC's optimizers
are doing?

e.g.

* "Why isn't this loop being vectorized?"

* "Did this function get inlined?  Why? / Why not?"

etc


What this talk is about
=======================

* GCC's current solution for this

* Problems with the current solution

* Improvements I've made for GCC 9

* Improvements I want to make

* Other ideas


Measure!
========

* Need to figure out what's slow first

* But: we have a disconnect between profiling and our optimization dumps

* Optimization dumps only talk about source file/line/column

* What if a function gets inlined in many different places?

* What about templates that are instantiated many times?


How does the user get *actionable* information?
===============================================

* "How do I make my code faster?"

  * "What stopped the loop getting vectorized?"

    * "Can I tweak it?"

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

  * can see multiple passes at once

  * granularity is still per-TU

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

.. nextslide::
   :increment:

It's easier to see with ``-fopt-info-missed``:

.. code-block:: none

  demo.cc:7:24: note: step unknown.
  demo.cc:7:24: note: vector alignment may not be reachable
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: no array mode for V2DI[3]
  demo.cc:7:24: note: Data access with gaps requires scalar epilogue loop
  demo.cc:7:24: note: can't use a fully-masked loop because the target doesn't have the appropriate masked load or store.
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.
  demo.cc:7:24: note: not ssa-name.
  demo.cc:7:24: note: use not simple.

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: no array mode for V2DI[3]
  demo.cc:7:24: note: Data access with gaps requires scalar epilogue loop
  demo.cc:7:24: note: op not supported by target.
  demo.cc:7:24: note: not vectorized: relevant stmt not supported:  _15 = _14 /[ex] 4;
  demo.cc:7:24: note: bad operation or unsupported loop bound.
  demo.cc:7:24: note: not vectorized: no grouped stores in basic block.
  demo.cc:7:24: note: not vectorized: no grouped stores in basic block.
  demo.cc:9:10: note: not vectorized: not enough data-refs in basic block.

i.e.:

.. code-block:: none

  demo.cc:7:24: note: not vectorized: relevant stmt not supported:
  _15 = _14 /[ex] 4;

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc:7:24: note: not vectorized: relevant stmt not supported:
  _15 = _14 /[ex] 4;

So we know that the failure is due to a (then) unsupported tree code
(fixed 2018-07-18 as of r262854).

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


What I've done so far for GCC 9
===============================

``-fsave-optimization-record``
==============================

* New in GCC 9

* machine-readable output format

* writes a ``demo.cc.opt-record.json`` file

Example of JSON output
======================

A tuple.

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

https://dmalcolm.fedorapeople.org/gcc/2018-05-16/preso-example-inlined/

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
``source_location`` (aka ``location_t``).

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


``AUTO_DUMP_SCOPE`` and ``DUMP_VECT_SCOPE``
===========================================

Replace all the:

.. code-block:: c++

   if (dump_enabled_p ())
     dump_printf_loc (MSG_NOTE, vect_location,
                      "=== vect_analyze_data_ref_accesses ===\n");

with just:

.. code-block:: c++

   DUMP_VECT_SCOPE ("vect_analyze_data_ref_accesses");

This captures that this ``note`` is expressing a frame somewhere on the
call stack during the optimization.

.. nextslide::
   :increment:

Textual output now indents them:

.. code-block:: none

  demo.cc:7:24: note: === analyze_loop_nest ===
  demo.cc:7:24: note:  === vect_analyze_loop_form ===
  demo.cc:7:24: note:   === get_loop_niters ===
  demo.cc:7:24: note:  Symbolic number of iterations is (((((unsigned long) _10 - (unsigned
  long) _9) - 24) /[ex] 8) * 768614336404564651 & 2305843009213693951) + 1


The JSON output nests all of these (and the notes within them), expressing
the hierarchy.


Future work
===========

We have about 2 months left for feature-development on GCC 9


What *should* the user experience be?
=====================================

"GCC can't vectorize <LOOP> because of <STMT>"

* what command-line options does the user provide?

* what output do they see?

.. nextslide::
   :increment:

What's important to the user?

Presumably:

* which loop?

* which statement, exactly, is stopping vectorization?


UX Ideas - input
================

* Currently the user can filter on:

  * what kind of pass ("ipa", "loop", "inline", "omp", "vec", plus "optall")

    e.g. ``-fopt-info-vec`` or somesuch ("tell me about vectorization")

  * what kind of message ("optimized", "missed", "note", "all")

* Or look at everything in one pass (for every function in the TU)
  e.g. ``-fdump-tree-vect``

.. nextslide::
   :increment:

* Things the user can't filter on yet:

  * just a particular function or range of source code
    (e.g. via a ``#pragma`` ?  via an attribute?)

  * based on hotness ("only tell me about code with a profile count above
    $THRESHOLD")

Current output
==============

(recall the two pages of "note" lines emitted at the loop's location)


UX Ideas - output
=================

* How about

  .. code-block:: none

    <LOOP-LOCATION>: couldn't vectorize this loop
    <PROBLEM-LOCATION>: because of <REASON>

.. nextslide::
   :increment:

* Use other prefixes than just "note" for everything

  * "missed-optimization" vs "optimization" ?

Rather than:

.. code-block:: none

     demo.cc:7:24: note: couldn't vectorize loop
     stl_vector.h:870:50: note: the reason

Maybe:

.. code-block:: none

     demo.cc:7:24: missed-optimization: couldn't vectorize loop
     stl_vector.h:870:50: note: the reason

(may require some DejaGnu tweaks)

.. nextslide::
   :increment:

* put the messages through the diagnostic subsystem

  * show source code

  * show inlining chain / inclusion chain etc

  * maybe show metadata:

    * "which pass?"

    * hotness

.. nextslide::
   :increment:

.. code-block:: none

  demo.cc: In function ‘std::size_t f(const std::vector<std::vector<float> >&)’:
  demo.cc:7:24: missed-optimization: couldn't vectorize loop due to...
  7 |   for (auto const & w: v)
    |                        ^
  In file included from ../x86_64-pc-linux-gnu/libstdc++-v3/include/vector:64,
                   from demo.cc:1:
  In function ‘std::vector<_Tp, _Alloc>::size_type std::vector<_Tp, _Alloc>::size() const
  [with _Tp = float; _Alloc = std::allocator<float>]’,
      inlined from ‘std::size_t f(const std::vector<std::vector<float> >&)’
      at demo.cc:8:18:
  ../x86_64-pc-linux-gnu/libstdc++-v3/include/bits/stl_vector.h:870:50:
  note:   not vectorized: relevant stmt not supported: _15 = _14 /[ex] 4;
  870 |       { return size_type(this->_M_impl._M_finish - this->_M_impl._M_start); }
      |                          ~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~

.. nextslide::
   :increment:

Idea for implementing the above:

* provide a way to filter out *most* of the output

  * everything apart from:

    * <LOOP-LOCATION>: "couldn't vectorize loop"

    * <REASON-LOCATION: "$REASON"

  * everything else counts as "details"

  * maybe filter out "details" by default, so that users by default get
    readable output

    * GCC developers (and DejaGnu tests) can opt-in to get the full details

.. nextslide::
   :increment:

Implementation ideas:

* explicitly mark the dump calls, adding a new flag(s) to the ``dump_*``
  API?

  * e.g. add ``MSG_DETAILS`` and add to *lots* of ``dump_`` calls

    * ugh (lots of churn)

  * e.g. add ``MSG_PRIORITY`` and add to a few ``dump_`` calls

* implicit: treat the messages that are in nested scopes as being "details",
  and filter them out implicitly

  * or add an ``auto_hide_dump_messages`` class, or somesuch

  * minimal patching required, some "magic"


``opt_result`` and ``opt_problem``
==================================

* a way to "bubble up" information about an optimization problem
  from deep in the call stack up to the top of a pass

* a special wrapper around `bool`

  * identifies the places that need failure-handling via the C++
    type system

    * both to the compiler, and to human readers

  * if ``dump_enabled_p``, has a "reason" string as well as the `false`
    `bool` - "why did it fail?"

.. nextslide::
   :increment:

Rather than:

.. code-block:: c++

     if (!check_something ())
       {
         if (dump_enabled_p ())
           dump_printf_loc (MSG_MISSED_OPTIMIZATION, vect_location,
                            "foo is unsupported.\n");
         return false;
       }
     [...lots more checks...]

     // All checks passed:
     return true;

.. nextslide::
   :increment:

we (optionally) capture the cause of the failure via:

.. code-block:: c++

     if (!check_something ())
       return opt_result::failure_at (stmt, "foo is unsupported");

     [...lots more checks...]

     // All checks passed:
     return opt_result::success ();

(this motivated the ATTRIBUTE_GCC_DUMP_PRINTF change above)

.. nextslide::
   :increment:

.. code-block:: c++

   return opt_result::success ();

is effectively the same as:

.. code-block:: c++

     return true;

but documents our intentions.

.. nextslide::
   :increment:

.. code-block:: c++

   return opt_result::failure_at (stmt, "foo is unsupported");

when ``!dump_enabled_p``, this is almost the same as:

.. code-block:: c++

   return false;

Fixing all those "problem locations" naturally fall
out of fixing the type issues needed to get it to compile

.. nextslide::
   :increment:

e.g. the specific failure case from our example was:

.. code-block:: c++

   if (!ok)
    {
      if (dump_enabled_p ())
        {
          dump_printf_loc (MSG_MISSED_OPTIMIZATION, vect_location,
                           "not vectorized: relevant stmt not ");
          dump_printf (MSG_MISSED_OPTIMIZATION, "supported: ");
          dump_gimple_stmt (MSG_MISSED_OPTIMIZATION, TDF_SLIM,
                            stmt_info->stmt, 0);
        }
      return false;
    }

(note the use of ``vect_location``)

.. nextslide::
   :increment:

and this becomes:

.. code-block:: c++

   if (!ok)
     return opt_result::failure_at (stmt_info->stmt,
                                    "not vectorized:"
                                    " relevant stmt not supported: %G",
                                    stmt_info->stmt);

(note the use of ``stmt_info->stmt``)

.. nextslide::
   :increment:

Status: am working on this; I hope to get it into gcc 9.

* fixes a lot of the issues discussed so far

* TODO: figure out that "details" vs "non-details" issue


"Why wasn't a pass run?"
========================

* "Was $PASS run?  Why not?"

* "Does $PASS get run at ``-O2``?"

End-users seem to have a lot of difficulty with this.

Non-trivial interaction of:

  * command-line options

  * ``opt_pass::gate`` virtual functions, and

  * the ``default_options_table`` (in ``opts.c``)

.. nextslide::
   :increment:

Idea/RFC:

* convert ``opt_pass::gate`` to return an ``opt_result`` rather than
  just a ``bool``

* hence, when dumping is enabled, we can tell the user
  *why* a pass wasn't run on a given function.

This would allow us to emit e.g.:

.. code-block:: none

  note: optimization pass 'vect' disabled for function 'foo'
  note: pass is enabled via '-ftree-loop-vectorize', or at '-O3' and above

(maybe as part of ``-fopt-info-vec-missed`` ?)


UX idea: Rich vectorization hints
=================================

Work-in-progress/prototype.

A much more verbose idea: "actionable" reports for the end-user.

* Rich optimization hints can contain a mixture of:

  * text

  * diagrams

  * highlighted source locations/ranges,

  * proposed patches

  * etc.

.. nextslide::
   :increment:

* can be printed to stderr (diagrams become ASCII art)

* can be saved as part of the JSON optimization record

  * can be prioritized by code hotness

  * browsed in an IDE, etc

  * diagrams become HTML/SVG?

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

Status:

* a hackish prototype

* lots of work remains; perhaps too much to get done for gcc 9


Summary
=======

* state of optimization dumps in gcc 8

* changes so far in gcc 9

  ``-fsave-optimization-record``

* proposed changes for gcc 9

  * filtering vectorization messages to just the most important, by default

  * showing better locations for problematic constructs

  * using the diagnostics subsystem?

  * show why a pass wasn't run?

* rich vectorization hints??


Questions and Discussion
========================

Thanks for listening!

(thanks to Red Hat for funding this work)

URL for these slides: https://dmalcolm.fedorapeople.org/presentations/cauldron-2018/

Source code for these slides: https://github.com/davidmalcolm/2018-cauldron-talk
