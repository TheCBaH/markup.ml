# Markup.ml &nbsp; [![version 0.5][version]][releases] [![(BSD license)][license-img]][license]

[version]:       https://img.shields.io/badge/version-0.5-blue.svg
[license-img]:   https://img.shields.io/badge/license-BSD-blue.svg

Markup.ml is a pair of parsers implementing the HTML5 and XML specifications.
Usage is simple, because each parser is just a function from byte streams to
parsing signal streams.

HTML5 gives complicated rules for well-formed markup, error reporting, and
recovery. Markup.ml encapsulates them. While the XML specification does not
include error recovery, Markup.ml also recovers from XML errors, after reporting
them. Thus, it provides best-effort parsing of both HTML and XML.

Here is an example of Markup.ml correcting errors in a small HTML fragment, then
pretty-printing it. The code is in the left column, and the center column shows
the values produced.

```ocaml
open Markup;;

string s                "<p><em>Markup.ml<p>rocks!"    (* malformed HTML *)

|> parse_html           `Start_element "p"
                        `Start_element "em"
                        `Text ["Markup.ml"]
                        ~report (1, 4) (`Unmatched_start_tag "em")
                        `End_element                   (* /em: recovery *)
                        `End_element                   (* /p: not an error *)
                        `Start_element "p"
                        `Start_element "em"            (* recovery *)
                        `Text ["rocks!"]
                        `End_element                   (* /em *)
                        `End_element                   (* /p *)
|> drop_locations
|> pretty_print         (* adjusts the `Text signals *)

|> write_html
|> to_string;;          "<p>
                           <em>Markup.ml</em>
                         </p>
                         <p>
                           <em>rocks!</em>
                         </p>"                         (* valid HTML *)
```

In addition to being error-correcting, the parsers are:

- *streaming*: capable of parsing partial input while more input is still being
  received;
- *lazy*: not parsing input unless it is needed to emit the next parsing signal,
  so you can easily stop parsing partway through a document;
- *non-blocking*: they can be used with [Lwt][lwt], but still provide a
  straightforward synchronous interface for simple usage; and
- *one-pass*: memory consumption is limited since the parsers don't build up a
  document representation, nor buffer input beyond a small amount of lookahead.

The parsers detect character encodings automatically. Strings emitted are in
UTF-8.

The parsers are subjected to fairly thorough [testing][tests], with more tests
to be added in the future.

## Interface and simple usage

The interface is centered around four functions between byte streams and signal
streams: [`parse_html`][parse_html], [`write_html`][write_html],
[`parse_xml`][parse_xml], and [`write_xml`][write_xml]. These have several
optional arguments for fine-tuning their behavior. The rest of the functions
either input or output byte streams, or transform signal streams in some
interesting way.

Some examples:

```ocaml
(* Show up to 10 XML well-formedness errors to the user. Stop after
   the 10th, without reading more input. *)
let report =
  let count = ref 0 in
  fun location error ->
    error |> Error.to_string ~location |> prerr_endline;
    count := !count + 1;
    if !count >= 10 then raise_notrace Exit

string "some xml" |> parse_xml ~report |> drain

(* Load HTML into a custom document tree data type. *)
type html = Text of string | Element of string * html list

file "some_file"
|> parse_html
|> tree
  ~text:(fun ss -> Text (String.concat "" ss))
  ~element:(fun (_, name) _ children -> Element (name, children))
```

## Advanced: Cohttp + Markup.ml + Lambda Soup + Lwt

The code below is a complete program that requests a Google search, then
performs a streaming scrape of result titles. The first GitHub link is printed,
then the program exits without waiting for the rest of input. Perhaps early exit
is not so important for a Google results page, but it may be needed for large
documents. Memory consumption is low because only the `h3` elements are
converted into DOM-like trees.

```ocaml
open Lwt.Infix

let () =
  Markup_lwt.ensure_tail_calls ();    (* Workaround for current Lwt :( *)

  Lwt_main.run begin
    Uri.of_string "https://www.google.com/search?q=markup.ml"
    |> Cohttp_lwt_unix.Client.get
    >|= snd                           (* Assume success and get body. *)
    >|= Cohttp_lwt_body.to_stream     (* Now an Lwt_stream.t. *)
    >|= Markup_lwt.lwt_stream         (* Now a Markup.stream. *)
    >|= Markup.strings_to_bytes
    >|= Markup.parse_html
    >|= Markup.drop_locations
    >|= Markup.elements (fun name _ -> snd name = "h3")
    >>= Markup_lwt.iter begin fun h3_subtree ->
      h3_subtree
      |> Markup.write_html
      |> Markup_lwt.to_string
      >|= Soup.parse
      >|= fun soup ->
        let open Soup in
        match soup $? "a[href*=github]" with
        | None -> ()
        | Some a ->
          a |> texts |> List.iter print_string;
          print_newline ();
          exit 0
    end
  end
```

This prints `aantron/markup.ml · GitHub`. To run it, do:

```sh
ocamlfind opt -linkpkg -package lwt.unix -package cohttp.lwt \
    -package markup.lwt -package lambdasoup scrape.ml && ./a.out
```

You can get all the necessary packages by

```sh
opam install lwt cohttp lambdasoup markup
```

## Installing

Until Markup.ml is added to OPAM, the easiest way to install it is by cloning
this repository, then running

```sh
make install
```

in the cloned directory. This will use OPAM to pin Markup.ml, install the
dependency Uutf, then build and install Markup.ml. If you want to use the module
`Markup_lwt`, check that Lwt is installed before installing Markup.ml.

To remove the pin later, run `make uninstall`.

## Documentation

The interface of Markup.ml is three modules [`Markup`][Markup],
[`Markup_lwt`][Markup_lwt], and [`Markup_lwt_unix`][Markup_lwt_unix]. The last
two are available only if you have Lwt installed.

## Help wanted

Parsing markup has more applications than one person can easily think of, which
makes it difficult to do exhaustive testing. I would greatly appreciate any bug
reports.

Although the parsers are in an "advanced" state of completion, there is still
considerable work to be done on standard conformance and speed. Again, any help
would be appreciated.

I have much more experience with Lwt than Async, so if you would like to create
an Async interface, it would be very welcome.

Please see the [CONTRIBUTING][contributing] file.

Feel free to open any issues on GitHub, or send me an email at
[antonbachin@yahoo.com][email].

[![Travis status][travis-img]][travis] [![Coverage][coveralls-img]][coveralls]

[travis]:        https://travis-ci.org/aantron/markup.ml/branches
[travis-img]:    https://img.shields.io/travis/aantron/markup.ml/master.svg
[coveralls]:     https://coveralls.io/github/aantron/markup.ml?branch=master
[coveralls-img]: https://img.shields.io/coveralls/aantron/markup.ml/master.svg

## License

Markup.ml is distributed under the BSD license. See [LICENSE][license].

The Markup.ml source distribution includes a copy of the HTML5 entity list,
which is distributed under the W3C document license. The copyright notices and
text of this license are also found in [LICENSE][license].

[releases]:        https://github.com/aantron/markup.ml/releases
[parse_html]:      http://aantron.github.io/markup.ml/#VALparse_html
[write_html]:      http://aantron.github.io/markup.ml/#VALwrite_html
[parse_xml]:       http://aantron.github.io/markup.ml/#VALparse_xml
[write_xml]:       http://aantron.github.io/markup.ml/#VALwrite_xml
[HTML5]:           https://www.w3.org/TR/html5/
[XML]:             https://www.w3.org/TR/xml/
[tests]:           https://github.com/aantron/markup.ml/tree/master/test
[signal]:          http://aantron.github.io/markup.ml/#TYPEsignal
[lwt]:             http://ocsigen.org/lwt/
[license]:         https://github.com/aantron/markup.ml/blob/master/doc/LICENSE
[contributing]:    https://github.com/aantron/markup.ml/blob/master/doc/CONTRIBUTING.md
[email]:           mailto:antonbachin@yahoo.com
[Markup]:          http://aantron.github.io/markup.ml
[Markup_lwt]:      http://aantron.github.io/markup.ml/Markup_lwt.html
[Markup_lwt_unix]: http://aantron.github.io/markup.ml/Markup_lwt_unix.html
