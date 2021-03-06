Non-blocking IO interface design for OCaml
------------------------------------------

The design should have the following properties:

1. Unified interface for blocking and non-blocking mode.
2. The existence of the non-blocking mode should not significantly
   impact blocking mode users.
3. Input possible from `in_channel`, `bytes`, refillable fixed-size `bytes`
   buffer (non-blocking mode).
4. Output possible to `out_channel`, `Buffer.t`, flushable fixed-size `bytes`
   buffer (non-blocking mode).
5. No third-party IO libraries/paradigms so that the module can adapt
   to the one the user chooses.
6. Reasonably efficient.

Suppose we want to IO streams of value of `type t`. For example xmlm's
signals (lexemes as they should be called) if you are familiar with
that.

For input (decoding) we begin with a type for input sources, decoders
and a function to create them.

    type src = [ `Channel of in_channel | `Bytes of bytes | `Manual ]
    type decoder
    val decoder : [< src] -> decoder

A `` `Manual`` source means that the client will provide the decoder with
chunks of bytes to decode at his own pace. The function for decoding
is :

    val decode : decoder -> [ `Await | `End | `Error of e | `Yield of t ]

`decode d` is :

- `` `Await`` iff `d` has a `` `Manual`` input source and awaits for
  more input. The client must use `Manual.src` (see below) to provide it.
- `` `Yield v``, if a value `v` of type `t` was decoded.
- `` `End``, if the end of input was reached.
- `` `Error e``, if an error `e` occured. If you are interested in a
  best-effort decoding, you can still continue to decode after
  the error.

For `` `Manual`` sources the function `Manual.src` is used to provide
the byte chunks to read from :

    val Manual.src : decoder -> bytes -> int -> int -> unit

`Manual.src d s k l` provides `d` with `l` bytes to read, starting at
`k` in `s`. This byte range is read by calls to `decode` with `d`
until `` `Await`` is returned. To signal the end of input call the function
with `l = 0`.

That's all what is needed for input. Just a note on the `` `Error``
case. Decoders should report any decoding errors with `` `Error`` to
allow standard compliant decodings. However at that point they should
give the opportunity to the client to continue to perform a best
effort decoding. In that case `decode` should always eventually return
`` `End`` even if `` `Error``s were reported before. I think best-effort
decoding on errors is a good thing: I was annoyed more than once with
xmlm simply failing with `` `Malformed_char_stream`` on files produced by
legacy software that gave invalid UTF-8 encodings for funky
characters. Rather than fail and block the client at that point it's
better to report an error and let it continue if it wishes so by
replacing the invalid byte sequence with U+FFFD.

For output (encoding) we begin with a type for output destinations,
encoders and a function to create them.

    type dst = [ `Channel of out_channel | `Buffer of Buffer.t |  `Manual ]
    type encoder
    val encoder : [< dst] -> encoder

A `` `Manual`` destination means that the client will provide to the
decoder the chunks of bytes to encode to at his own pace. The function
for encoding is :

    val encode :
      encoder -> [< `Await | `End | `Yield of t ] -> [ `Ok | `Partial]

`encode e v` is

- `` `Partial`` iff `e` has a `` `Manual`` destination and needs more output
  storage. The client must use `Manual.dst` (see below) to provide it and
  then call ``encode e `Await`` until `` `Ok`` is returned.
- `` `Ok`` when the encoder is ready to encode a new `` `Yield`` or `` `End``.

Raises `Invalid_argument` if a `` `Yield`` or `` `End`` is encoded after a
`` `Partial`` encode (this is done to prevent the encoder from having
to bufferize `` `Yield``s).

For `` `Manual`` destinations the function `Manual.dst` is used to provide
the byte chunks to write to :

    val Manual.dst : encoder -> bytes -> int -> int -> unit

`Manual.dst e s k l` provides `e` with `l` bytes to write, starting at
`k` in `s`. This byte range is written by calls to `encode` with `e`
until `` `Partial`` is returned. To know the remaining number of
non-written free bytes in `s` the function `Manual.dst_rem` can be
used:

    val Manual.dst_rem : encoder -> int

`Manual.dst_rem e` is the remaining number of non-written, free bytes
in the last buffer provided with `Manual.dst`. A well-behaved encoder
should always fill all the bytes it is given, except for the buffer
that encodes the `` `End``.

One note on `` `Manual`` destinations, encoding `` `End`` always
returns `` `Partial``. The client should then as usual use `Manual.dst`
and continue with `` `Await`` until `` `Ok`` is returned at which
point `Manual.dst_rem e` is guaranteed to be the size of the last
provided buffer (i.e. nothing was written, this is a good property for
the client's loops).
