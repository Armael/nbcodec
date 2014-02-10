Nbcodec — non-blocking IO interface design for OCaml
-------------------------------------------------------------------------------

In `src/se.ml` there are two implementations of a streaming codec for
simplified ASCII s-expressions.

* `Se.B` implements a blocking codec.
* `Se.Nb` implements a non-blocking codec.

The api documentation can be generated with 

    ./build doc

the result is in `_build/doc/api.docdir/Se.html`. 

The executable `setrip.native` can be used to test and benchmark the codecs, 
the source is `test/setrip.ml` it can be built with :

    ./build setrip.native

For example : 

  ./setrip.native -enc -rseed 1067894368 > 1067894368.sexp 

will generate a 32 Mo file of random s-expressions with the non-blocking
codec. Use `setrip.native -help` for more information.